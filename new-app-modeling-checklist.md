# Aerospike New App Modeling Checklist

**Summary:** Mandatory pre-implementation checklist for modeling a new Aerospike-backed application. Use this before writing schemas, APIs, or code.

**Status:** Normative guide for this research folder. Use alongside [README.md](README.md) and [concepts-and-patterns.md](concepts-and-patterns.md).

---

## 0) Required outputs before implementation

Produce and review all items below before implementation starts:

| Output | Required content |
|--------|------------------|
| **Domain entity and relationship map (first)** | Core entities, relationship types (1:1 / 1:N / N:M), ownership boundaries, lifecycle notes, cardinality ranges, skew assumptions. |
| **Access pattern matrix (second)** | Every read/write path on those entities, frequency, latency target, key or query path, expected payload size. |
| **Key schema** | Namespace, set, key format, key examples, cardinality estimate, skew notes. |
| **Bin schema** | Bin names, types, constraints, size ranges, growth model, update frequency, ownership. Each bin must pass the bin-extraction checkpoint (4.1) and per-bin justification gate (4.2). |
| **Example records (JSON)** | One JSON example per set, showing realistic values for all bins. Placed inline immediately after each set's key and bin schema tables. Serves as a concrete reference for reviewers and implementers — makes the record shape unambiguous at a glance. |
| **Relationship decisions** | 1:1 / 1:N / N:M mapping, owner side, consistency strategy, delete/cascade plan. |
| **Pattern decision forms (per major 1:N or N:M)** | Completed deterministic consolidation-vs-split worksheet with required inputs, contention math, decision band, and explicit tie-break/exception notes. |
| **Sizing decision worksheet** | For each major 1:N: child size distribution (min/p50/p95/max), child-count distribution per parent (min/p50/p95/p99/max), aggregate bytes per parent (p95), chosen pattern and explicit "why not" alternatives. |
| **Index rationale** | Why each SI/set index/expression index exists, expected selectivity, memory impact. |
| **Deployment size constraints** | The target namespace's configured **`max-record-size`**, stated explicitly. This bounds every consolidation decision in this checklist, so it is an input, not an outcome. If it cannot be obtained, record `max-record-size: 1 MiB (ASSUMED — server default)` as an approved assumption with the reconsider trigger *"confirm before implementation; if the namespace is configured lower, re-run the sizing worksheet."* Never assume the 8 MiB architectural ceiling. |
| **Growth and hot-key plan** | Record growth limits, split or overflow trigger, sharding strategy for hot keys. |
| **Validation plan** | Synthetic workload tests, edge-case tests, failure-mode tests, acceptance thresholds. |

If any item is missing, the data model is not ready for implementation.

---

## 0.1) Mandatory clarification gate (before modeling)

Before drafting any concrete data model, explicitly ask clarifying questions for any ambiguity in:

- **PRD scope and invariants** (what is in/out of scope, immutable rules, delete semantics).
- **Entity definitions and lifecycle ownership** (who can create/update/delete each object, placeholder vs hard delete behavior).
- **Access patterns and correctness/latency expectations** (read/write paths, ordering, pagination, consistency level). If composite IDs use `created_at_canonical` or similar placeholder terms, require explicit resolution of the string format (encoding, precision, timezone, allowed characters, sort behavior, collision suffix policy) in this gate. Do not proceed with an unresolved canonical format. (Note: `created_at_canonical` is an ID-recipe variable, not a bin name — see [id-selection-guidance.md](id-selection-guidance.md) section 6.) When multiple read shapes compete for "dominant" on the same entity group (e.g., root-thread reads AND single-node reads for comments), require the stakeholder to provide approximate read-share percentages. If unavailable, mark `primary_read_shape` as `BLOCKED_MISSING_INPUT` rather than choosing subjectively. **Fallback for autonomous modeling:** If read-share percentages are unavailable and cannot be obtained (autonomous modeling pass, stakeholder unreachable), apply the **access-pattern inference default** — classify the read shape with the strongest structural signal (the one that determines record layout, typically the shape used by the dominant display surface) as primary. Document the inferred split as an `ASSUMPTION` with a reconsider trigger: *"If measured read traffic shows the secondary shape exceeds N% of reads, reassess pattern choice."* This allows autonomous modeling to proceed without blocking while preserving an explicit audit trail.
- **Object growth assumptions** (typical/max cardinality, skew/outliers, growth over time, fan-out risk).
- **Deployment size constraint** — the target namespace's configured **`max-record-size`**. Every consolidation-vs-split decision downstream is bounded by it, and the server default (1 MiB) is far below the 8 MiB architectural ceiling that models are often mistakenly sized against. Ask for the configured value. If it is unavailable, proceed under the documented default as an approved assumption rather than blocking, and flag it for confirmation before implementation.

Rule: do not wait for the user to prompt for these. If ambiguity exists, ask first; then model.

Question quality rule:

- Ask requirement-gap questions, not implementation-preference questions.
- Prefer behavior/usage wording (frequency, growth/skew, consistency tolerance, lifecycle requirements).
- Infer what you can from requirements and local research; ask only for irreducible missing inputs.
- If a deterministic rubric/default resolves a choice, present the default with rationale and ask only for exception confirmation.

---

## 0.2) How to use this checklist

### Workflow order

Follow these steps in sequence. Steps 3–5 repeat for each entity group — one group at a time.

1. **Clarification gate (0.1)** — Resolve broad ambiguities: PRD scope, consistency model, delete strategy, ID format fork. Group-specific sizing and cardinality questions are deferred to the per-group pass.
2. **Domain entities and relationships (1)** — Define what exists: entities, relationships, cardinality, ownership.
3. **Access pattern matrix (2)** — Define how each entity is read and written: operations, latency, payload, correctness level.
4. **Entity-group plan** — Partition entities into groups that share relationships. For each group, list applicable packs and input status (confirmed vs missing). This plan is the checkpointing artifact — each group carries a status. See README Step 2.5.
5. **Per entity group** (repeat for each group in plan order):
   - (a) **Group-specific clarification** — Resolve remaining inputs for this group's decision packs.
   - (b) **Record design** — Key schema (3), bin schema (4, 4.1, 4.2), and relationship decisions (5) for this group. Route each relationship to its pack (5.0.1), run the shared sizing gate (5.1) and contention rubric (5.2), then complete the applicable pack (5.4–5.8). After packs, check whether any access pattern requires a cross-entity derived metric (5.8.1).
   - (c) **Developer walkthrough (5.9)** — Trace 2–3 scenarios through this group's schema.
   - (d) **Implementer review (5.9.2)** — Submit schemas and walkthroughs for review by someone with an implementer's perspective. Triage findings.
   - (e) **Stakeholder review checkpoint (5.9.3)** — Present schemas, assumptions log, walkthrough results, implementer review findings, and open items. Do not proceed until approved.
   - (f) **Update plan** — Mark group status: `done`, `blocked`, or `revision needed`.
6. **Index strategy (6)** — Justify every secondary index; prefer key-based access.
7. **Version gates (7)** — Lock DB/client versions and define fallbacks.
8. **Sizing/growth guardrails (8)** — Set numeric thresholds for record size, overflow, and hot keys.
9. **Validation plan (9)** — Define synthetic workload tests, failure-mode tests, and acceptance thresholds.
10. **LLM guardrails (10)** — Apply when an LLM is performing the modeling (ID selection, timestamps, default discipline, fabrication prevention).
11. **Final readiness gate (11)** — Verify all outputs exist and pass acceptance checks.

**Anti-pattern: single-pass execution.** Steps 3–5 are an interactive loop, not a batch operation. Each entity group requires group-specific clarification (5a), record design with sizing and contention gates (5b), a developer walkthrough (5c), an implementer review (5d), and a stakeholder review checkpoint (5e) — in that order, completed for one group before starting the next. Do not plan to execute all groups in a single pass or pre-decide pattern choices for later groups while working on earlier ones. See section 10.15 for the LLM-specific execution cadence guardrail.

### Shared gates in section 5

Sections 5.0.2 (server-side operation preference), 5.1 (sizing gate), and 5.2 (contention rubric) are **shared infrastructure**, not pack-specific steps. They sit before the packs so you can internalize them once, then apply them to each relationship as you enter the relevant pack. Every pack (5.4–5.8) consumes 5.1 and 5.2, and 5.0.2 applies as a verification step after sizing and contention favor consolidation — you do not need to scroll back to re-read them for each pack, but you do need to run them for each new relationship.

The 60-second triage (5.3) is an optional fast pre-check; it does not replace the full gates.

### Normative defaults vs project overrides

This checklist operates at two layers:

- **Normative defaults** — What the checklist prescribes when no project-specific input overrides the choice (e.g., "consolidated hierarchy is the default when Band A conditions hold," "default to UUIDv4 for hash IDs"). These are the starting positions.
- **Project-required overrides** — Stakeholder-confirmed inputs that change a default (e.g., "use cleartext composite IDs," "accept full fan-out at celebrity scale"). When a project input conflicts with a normative default, the project input wins per the conflict-resolution order in section 10.6. Log the override.

If project inputs are partially complete, apply normative defaults for resolved areas and mark unresolved areas `BLOCKED_MISSING_INPUT`. Do not guess which layer applies — the input either exists or it doesn't.

---

## 1) Domain entities and relationships (must come first)

Before discussing read/write paths, define what is being accessed:

- Entity list with concise definitions.
- Relationship map (1:1, 1:N, N:M) and directionality.
- Cardinality ranges (typical, max) and skew assumptions.
- Ownership and lifecycle boundaries (create/update/delete authority).
- Candidate consistency requirements by relationship (eventual vs transactional).

Rule: do not finalize access patterns until this map is complete.

---

## 2) Access pattern matrix (first-class artifact)

For each operation, capture:

- Operation name (e.g. "get user profile", "append event", "list followers page").
- Read/write type and expected QPS.
- Required latency (p50/p95/p99 if available).
- Access path: direct key, bounded batch, SI query, scan.
- Expected records touched and payload size.
- Correctness level: eventual, read-after-write, transactional.

Use this matrix to drive all record layout decisions.

---

## 3) Key schema contract

Define and freeze:

- Namespace and set per entity/workload.
- Record key format and examples.
- Key derivation rules (especially for sliced keys like account+bucket).
- Expected key cardinality and skew (long tail, celebrity users, bursty entities).
- Partition/hot-key risk assumptions and mitigation.

Rule: optimize key design for common reads to be single get or bounded batch get.

#### Set multiplicity for structurally identical records

When the same record schema (same bins, same CDT pattern, same access operations) applies to multiple parent entity types — for example, Twitter likes on tweets and likes on comments both store an ordered list of liker handles keyed by content ID — choose between a single shared set and per-parent-type sets.

| Approach | Mechanism | Advantages | Disadvantages |
|----------|-----------|------------|---------------|
| **Single set + key prefix** | One set (e.g., `likes`), key = `tweet:{id}` or `comment:{id}` | One set to declare and maintain; one SI definition covers all types; simpler mental model | Key-prefix convention must be documented and enforced; set-level operations (truncate, count, scan) cannot scope to one parent type |
| **Per-parent-type sets** | Separate sets (e.g., `likes_tweet`, `likes_comment`), key = `content_id` | Natural Aerospike set boundaries; set-level truncate/count/scan scopes per type; SI definitions can differ per type if needed | More sets to declare; SI must be defined per set if needed; mental overhead when the record schema is identical across sets |

**Decision heuristic:**

- If any access pattern, SI definition, TTL policy, or set-level operation (truncate, scan, count) needs to differ by parent type, use per-parent-type sets.
- If all access patterns, SIs, TTLs, and operations are identical across parent types, a single set with key prefix is simpler and sufficient.
- If uncertain, per-parent-type sets are the safer default — they preserve the option to diverge later without key-schema migration.

Document the choice and rationale in the set table.

---

## 4) Bin schema contract

For each bin:

- Name (<= 15 chars), type, and nested structure contract.
- Typical and max size contribution.
- Update pattern (append-heavy, random updates, counters, immutable).
- Query shape against this bin (value/range/rank/list membership).
- Compatibility and evolution notes (optional bins, default values, migration path).

Naming rule: prefer descriptive bin names by default. Only abbreviate when needed to fit the 15-character bin-name limit.

Prefer explicit bin contracts over "schemaless means anything."

After defining the key schema (section 3) and bin schema tables for a set, include a **JSON example record** showing realistic, populated values for every bin. The example must:

- Use the set's key format as the label: **Example record — `{set_name}` key `{key_example}`:**
- Show representative values (not empty or default) for all bins, including nested CDT structures (maps, lists, list-of-lists, list-of-maps).
- Be a fenced JSON code block placed directly after the bin schema table (before the bin-extraction checkpoint).

The example record is a required deliverable per set — it makes the record shape reviewable at a glance without mentally parsing table definitions.

### 4.1) Bin-extraction checkpoint (mandatory per-bin gate)

For each bin being added to an existing entity record, verify all three conditions before placement. This checkpoint applies during record design (Steps 3–5) and catches lifecycle mismatches that the relationship-level split signals in section 5 may miss.

1. **Write frequency** — Is this bin's write rate comparable to the host record's other bins? If it is significantly higher (e.g., append on every user action vs. infrequent profile edits), extract to a companion record.
2. **Retention/lifecycle** — Does this bin's data have the same retention as the host record? If the bin data is time-bounded (e.g., 14-day window) but the host record is permanent, extract to a companion record with appropriate expiry (record TTL, background scan/trim, or day-bucketing).
3. **Expiry mechanism** — If the data is time-bounded, what mechanism removes old entries? Document one of: record-level TTL, application-level trim on write, application-level trim on read, background scan/trim, or day-bucketed records with TTL. "No mechanism" is not acceptable for time-bounded data.

If any check fails, extract the bin to its own record/set and run the extracted structure through pack routing (section 5.0.1) independently.

If all three checks pass, co-location on the host record is the default. Do not extract a bin to a companion record solely for organizational uniformity — extraction has a cost (one additional PI entry per record, an extra read on any path that needs both the host and the companion). Extract only when a check fails or the sizing gate (5.1) shows the bin's growth trajectory will push the host record beyond its operational band.

**Worked example — `convo_ids` on user record.** A "My Conversations" feature requires a list of conversation root IDs where the user recently participated (14-day retention, append on every top-level comment). The user record is permanent and mostly stable.

- Write frequency: fails — appends on every comment vs. infrequent profile edits.
- Retention: fails — 14-day window vs. permanent.
- Expiry mechanism: no record-level TTL on user record; no per-element TTL in Aerospike lists.

**Resolution:** Extract to a companion record (`user_conversations`, key = `{handle}`). The companion enters pack routing as an event-timeline candidate (append-heavy, time-ordered, 14-day retention) and is subject to Pack 4 sizing math. Expiry: daily background scan/trim removing entries older than 14 days (prefix removal from an ordered list). The user record stays clean and stable.

### 4.2) Per-bin justification gate (mandatory for implementation-ready specs)

For each bin in the contract, document:

- **Functional purpose:** What user-visible behavior or API contract requires this bin?
- **Source anchor:** Which requirement, stakeholder answer, or access pattern drives it?
- **Alternatives considered:** Could this be served by Aerospike metadata (generation, LUT), another existing bin, or a computed value?
- **Removal test:** What breaks if this bin is removed?

If any bin cannot pass this gate, mark it as `OPTIONAL` or remove it from the contract.

**Acceptance rule:** No bin is contract-final unless it has a requirement-anchored purpose and a removal-impact statement.

**Timestamp-specific application:** Do not add multiple timestamp bins for the same object by default. Add a timestamp bin only when a specific requirement names the behavior it enables (display, API contract, lifecycle, compliance, analytics) and that behavior is not already covered by immutable business fields or Aerospike metadata. For read-modify-write correctness, use generation-based CAS (`EXPECT_GEN_EQUAL`), not timestamp-bin comparison. For record freshness introspection, use Aerospike LUT metadata unless a persisted timestamp is explicitly required by product/API contract.

---

## 5) Relationship and consolidation decisions

Use [concepts-and-patterns.md](concepts-and-patterns.md) and [one-to-many-relationships.md](one-to-many-relationships.md), then document the selected pattern and why.

**Growth behavior is the primary split signal.** Immutable fields, slowly-changing fields (updated in place), and small slowly-growing lists belong together on the same record by default. The reason to split data into a separate record is when a portion of the record grows by accumulation (list appends, map key additions) at a rate or to a size that would degrade performance or approach the namespace's configured `max-record-size`. Evaluate record size *trajectory*, not just current size — a 2 KiB list that never exceeds 5 KiB is not a split candidate; a 2 KiB list that grows by 100 bytes/day with no bound is.

### When consolidation is a good default

- Parent-driven access dominates.
- Child objects are small.
- PI cost of many tiny records would dominate.
- One-record read/write path materially reduces latency.

### When NOT to consolidate (required explicit check)

Choose split records (or split+index) when one or more are true:

- **Subset hot-path reads:** the common request needs only a small slice, but consolidation forces full-record transfer or large in-record scans.
- **Update-frequency asymmetry:** one sub-component updates very frequently and would cause write amplification on a large consolidated record.
- **Independent lifecycle/TTL:** related data has materially different retention, archival, or compliance requirements.
- **Contention risk:** many clients write to the same consolidated key, creating hot-key pressure.
- **Unbounded growth risk:** list/map growth can exceed healthy operating size or trend toward the configured `max-record-size`.
- **Inverse access dominates:** common reads are child->parent or "find parents containing X", better served by child-held references plus index.

Document the chosen split pattern and fallback trigger (for example, "if record exceeds X KiB, activate bucket/overflow pattern"). For the explicit companion overflow pattern (`subkeys` bin, filter-expression routing, bucket strategies), see [one-to-many-relationships.md](one-to-many-relationships.md) § 5.

### PI cost/benefit gate for auxiliary lookup sets

When introducing an auxiliary set whose sole purpose is to enable a secondary access path (e.g., a lookup table mapping one ID to another), evaluate the cost before adding it:

1. **Compute the PI cost:** `expected_record_count × 64 bytes`.
2. **Compare to alternatives:** SI query, application-provided context (e.g., the caller always knows the root ID from navigation context), bounded batch scan.
3. **Document:**
   - (a) the access path this set enables,
   - (b) the expected query frequency for that path,
   - (c) the PI cost at projected scale,
   - (d) why the alternative is worse.

If the PI cost exceeds the benefit — for example, 1.6 GB PI for a secondary read path that is always reachable through a primary path provided by the caller's navigation context — reject the auxiliary set.

**Worked example — `cm_root` lookup set for comment-by-ID.** A proposed set stores one record per comment globally, mapping `comment_id` → `content_root_id` to enable O(1) comment-by-ID lookup without knowing the root. At 25M comments: PI cost = `25M × 64 bytes = 1.6 GB`. The alternative: the caller navigating to a comment from a notification deep-link already has the content root ID in the notification payload; from a user profile's comment history, a `user_comments` companion record provides root IDs. If the independent comment-by-ID path is truly hot and cannot be served through navigation context, the PI cost may be justified — but the spec must document the frequency and show that the alternative is materially worse.

### 5.0) Read-shape glossary

Use these terms consistently when classifying `primary_read_shape` in decision packs and the contention rubric. Choose the term set that matches the relationship structure.

**Hierarchy contexts** (use when children can nest — tree-structured data like threaded comments, category taxonomies, org charts):

- **whole-tree** — Read loads the full tree from root through all nesting levels. Example: "Load post + full comment thread."
- **subtree** — Read loads a branch starting from an interior node through its descendants, not the whole tree. Example: "Load the reply chain under a specific comment."
- **single-node** — Read targets one node by its own identity, independent of tree context. Example: "Get comment by ID from a notification deep-link."
- **mixed** — No single shape dominates; require approximate read-share percentages from the stakeholder.

**Flat 1:N contexts** (use when children don't reference each other and have no tree structure — e.g., posts by a user, likes on a post, items in an order):

- **full-set** — Read loads the parent + all children. Example: "Load all posts by this user."
- **subset** — Read loads the parent + a filtered, projected, or paginated slice of children. Example: "Load the 10 most recent posts."
- **single-element** — Read targets one child by its own identity, independent of the parent. Example: "Load one post by post ID."
- **mixed** — No single shape dominates; require approximate read-share percentages from the stakeholder.

**Non-relationship contexts** (flat entity, no parent-child structure):

- **single-key read** — Record accessed directly by key. No traversal or parent-child concept applies.

**Contention mapping:** In consolidated designs, `whole-tree`, `subtree`, `full-set`, and `subset` all target the root/parent record for contention purposes. `single-node` and `single-element` may or may not hit the root record depending on whether children are consolidated or split — this distinction drives the contention rubric in 5.2.

### 5.0.1) Pack routing (required before entering a decision pack)

Before filling out a decision pack, determine which pack applies to each major relationship based on its structure:

- **Flat 1:N** (children don't reference each other, no nesting) → **Pack 1** (section 5.4). Use flat 1:N read-shape terms (`full-set`, `subset`, `single-element`).
- **N:M** (bidirectional, both sides are first-class entities) → **Pack 2** (section 5.5).
- **Tree-structured children** (children can nest: threaded comments, category hierarchies, nested folder structures) → **Pack 3** (section 5.6). Use hierarchy read-shape terms (`whole-tree`, `subtree`, `single-node`). Use Pack 3 if **any** child-to-child nesting exists, even if most children are top-level.
- **Flat reply chains (depth = 1, no child-to-child nesting):** Even when the domain calls these "threads" or "replies," if children cannot reference other children and nesting depth is strictly 1, the structure is flat 1:N → **Pack 1**, not Pack 3. Pack 3 applies only when children can nest (child-to-child references exist). Pack 4 applies when the data is retention-driven with append-heavy time-ordered semantics and independent TTL — not merely because replies are chronological.
- **Event timelines and time-bounded reference structures** (append-heavy, time-ordered, retention-driven) → **Pack 4** (section 5.7). This includes user-facing event feeds, notifications, **and** per-user time-bounded index/reference lists (e.g., recent conversations, activity logs) — any data structure with append-heavy writes and a retention window, regardless of whether it tracks incoming events or the user's own actions. The data characteristics (append-heavy, time-ordered, retention-driven) determine the pack, not the domain label.
- **Multi-record delete/update side effects** → **Pack 5** (section 5.8), in addition to the pattern pack above.

A single entity group may require multiple packs (e.g., comments use Pack 3 for hierarchy shape + Pack 5 for delete cascade cleanup). Complete all applicable packs.

### 5.0.2) Server-side operation preference (shared — applies to all packs)

Prefer record granularity where every declared read, write, query, and mutation maps to a concrete server-side Aerospike operation (single get, operate, SI query, batch get). Consolidation reduces PI cost and improves read locality, but it is a means to those ends, not an end in itself.

**Normal vs problematic client-side work.** Client-side assembly of multiple server results for display is normal — for example, batch-reading several day buckets and merging them into a feed page, or reading a comment tree and sorting root comments by rank. These are composition tasks that combine server-provided data. Client-side workarounds are different: set-difference logic for read/unread state, multi-step read-compute-write for mutations the server could handle at a different granularity, or client-side filtering to compensate for a storage layout that cannot serve a declared operation directly. When consolidated storage forces such workarounds, that is a signal to reconsider granularity.

**Bounded PI changes the calculus.** The PI savings from consolidation are most significant for permanent, accumulating records. For TTL-bounded data (event timelines, notifications, session state), the PI cost of a simpler granularity is capped: `event_rate_p99 × ttl_days × 64 bytes`, all auto-expiring. When the bounded PI cost is acceptable and the simpler granularity serves all declared operations as direct server-side calls, the PI savings from consolidation may not justify the operational complexity it introduces.

**Relationship to other gates.** This principle does not override the sizing gate (5.1) or contention rubric (5.2). It adds a verification step: after sizing and contention analysis favor consolidation, check that all declared operations still map cleanly to server-side calls. If they do not, document the client-side workarounds and evaluate whether the simpler granularity — with its higher but bounded PI cost — is the better tradeoff.

### 5.1) Mandatory sizing-to-pattern decision gate (shared — applies to all packs)

This gate is used by Packs 1–4 (sections 5.4–5.7). Run it for each major relationship before entering the applicable pack.

Before locking split-vs-consolidate for any 1:N, complete this gate:

1. Compute child-size distribution (min/p50/p95/max bytes).
2. Compute child-count distribution per parent (min/p50/p95/p99/max).
3. Compute projected aggregate per parent (at least p95 and p99).
4. Compare aggregate and write profile to explicit operational thresholds (not just the configured `max-record-size`).
5. Choose pattern and document:
   - chosen pattern (list-on-parent, consolidated parent record, child+SI, sharded/overflow),
   - why this pattern fits the measured distributions and access patterns,
   - why the rejected alternatives are worse for this workload.

If this gate is not documented, the model is not ready.

### 5.1.1) Example — blocked decision record

When required inputs are missing, the correct output is a partially-filled decision record with explicit gaps — not silence, and not fabricated estimates. Use this as a template:

> **7.6 Likes on Tweets/Comments**
> **Status: `BLOCKED_MISSING_INPUT`**
>
> Confirmed inputs:
> - Dominant read shape: content-driven (count display). Source: Data modeling brief.
> - Uniqueness: one like per user per item. Source: PRD.
>
> Missing inputs:
> - Like count per tweet (p50/p95/p99): **NOT IN REQUIREMENTS.** The sizing reference covers comments and retweets but not likes.
>   - **Question:** "What is the expected number of likes per tweet at p50, p95, and p99? Pattern selection between list-on-parent vs separate record depends on whether p99 likers fit on the content record without exceeding the Goldilocks band."
> - Like count per comment (p50/p95/p99): **NOT IN REQUIREMENTS.**
>   - **Question:** "What is the expected number of likes per comment at p50, p95, and p99?"
>
> **Tentative pattern: None selected. Awaiting input.**

This is what "stopping" looks like. It is a partially-filled record with explicit gaps and targeted questions — not an empty section, and not a section filled with invented numbers.

### 5.2) Deterministic consolidation-vs-split rubric (shared — required per major relationship)

This rubric is used by Packs 1–4 (sections 5.4–5.7). Run it for each major 1:N or N:M relationship before entering the applicable pack.

For every major 1:N or N:M decision, evaluate both candidate patterns:

1. consolidated-per-root record
2. split records (or sharded/hybrid variant)

Do not choose a pattern until this rubric is completed.

#### Required inputs

- `events_per_root_active_window` (expected count of writes per root in active window)
- `active_window_sec` (window duration in seconds)
- `contention_window_ms` (same-key contention window; default 8 ms, allowed 4-8 ms)
- `burst_factor_p99_over_avg` (default 10 if unknown)
- `avg_child_payload_bytes`
- `projected_child_count_p99`
- `projected_record_size_kib_p99`
- `growth_horizon_days`
- `write_slo_p95_ms`
- `primary_read_shape` (hierarchy: `whole-tree`, `subtree`, `single-node`, `mixed`; flat 1:N: `full-set`, `subset`, `single-element`, `mixed` — see glossary in 5.0)
- `delete_semantics` (placeholder, subtree delete, hard delete, etc.)
- `load_test_done` (`yes`/`no`)
- `observed_key_busy_rate`
- `observed_write_p95_ms`

#### Stop-and-ask gate (hard requirement)

If any required input is missing, stale, or low-confidence:

- stop pattern selection immediately,
- mark status `BLOCKED_MISSING_INPUT`,
- ask targeted clarification questions for missing fields,
- do not pick consolidate/split until inputs are complete.

Minimum targeted questions:

- What is p95/p99 event concentration per root (not only average)?
- Over what active window are those events concentrated?
- What write p95 and conflict/error budget must be met?
- What is projected p99 record size at the growth horizon?
- Is dominant read path whole-tree/full-set or single-node/single-element? (see glossary in 5.0)
- What delete/cascade behavior is mandatory?

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| `events_per_root_active_window` | Traffic analysis, sizing reference | "How many [child] writes per [root] are expected in the active window?" |
| `active_window_sec` | Traffic analysis, stakeholder | "Over what time window are [child] writes concentrated (seconds)?" |
| `contention_window_ms` | Ops/infra team, default 8 ms | "What is the same-key contention window for your Aerospike deployment?" |
| `burst_factor_p99_over_avg` | Traffic analysis, default 10 | "What is the p99-to-average burst ratio for [child] writes?" |
| `avg_child_payload_bytes` | Sizing reference, sample data | "What is the average [child] payload size in bytes?" |
| `projected_child_count_p99` | Sizing reference, traffic analysis | "What is the expected [child] count per [root] at p99?" |
| `projected_record_size_kib_p99` | Sizing worksheet (computed) | "What is the projected record size at p99 child count and payload?" |
| `growth_horizon_days` | Stakeholder, product roadmap | "Over what time horizon should the model remain healthy without re-architecture?" |
| `write_slo_p95_ms` | Stakeholder, SLA | "What is the required p95 write latency for [child] operations?" |
| `primary_read_shape` | Access pattern analysis | "Is the dominant read path whole-tree/full-set or single-node/single-element? (see glossary in 5.0)" |
| `delete_semantics` | PRD, stakeholder | "What delete/cascade behavior is mandatory for [root] and its [children]?" |
| `load_test_done` | Engineering team | "Has a load test been run for this workload? If so, what were the key-busy rate and write p95?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Required calculations

- `avg_qps = events_per_root_active_window / active_window_sec`
- `burst_qps = avg_qps * burst_factor_p99_over_avg`
- `lambda_window = burst_qps * (contention_window_ms / 1000)`

Approximate probability of two or more arrivals inside one contention window:

- `P(>=2) = 1 - exp(-lambda_window) * (1 + lambda_window)`

Interpret `lambda_window` as expected same-key arrivals per contention window.

#### Concise worked example (comments)

Use this as a reference pattern for applying the equation.

- Assumptions: `events_per_root_active_window = 350..590` comments, `active_window_sec = 14400` (4 hours), `contention_window_ms = 8`, `burst_factor_p99_over_avg = 10`.
- `avg_qps = 350/14400 .. 590/14400 = 0.024 .. 0.041`
- `burst_qps = avg_qps * 10 = 0.24 .. 0.41`
- `lambda_window = burst_qps * 0.008 = 0.0019 .. 0.0033`
- `P(>=2)` in one contention window is near zero at this lambda range.

Interpretation for this scenario: contention risk from same-key concurrent comment writes is very low, so consolidation is acceptable from a contention perspective. Final pattern choice must still pass record-size, SLO, and safety-check gates.

#### Decision bands (must follow first matching rule)

1. **Band A: Consolidate** — All of the following must hold:
   - `lambda_window < 0.05`
   - `projected_record_size_kib_p99 < 1024`
   - `observed_key_busy_rate < 0.1%`
   - `observed_write_p95_ms <= write_slo_p95_ms` (or no evidence of breach when load test pending)

   Consolidation is the normative default. No mandatory guardrails beyond standard overflow trigger documentation.

2. **Band B: Consolidate with guardrails** — Any one of the following triggers Band B:
   - `0.05 <= lambda_window < 0.2`, or
   - `1024 <= projected_record_size_kib_p99 < 4096`, or
   - `0.1% <= observed_key_busy_rate < 1%`

   Consolidation is allowed with mandatory guardrails (retries with jitter, telemetry, shard trigger).

3. **Band C: Split/sharded required** — Any one of the following triggers Band C:
   - `lambda_window >= 0.2`, or
   - `projected_record_size_kib_p99 >= 4096`, or
   - `observed_key_busy_rate >= 1%`, or
   - measured write p95 breaches SLO under expected burst load

**Classification rule:** Band assignment follows first-matching-rule order. If all Band A conditions hold, the entity is Band A regardless of how close the metrics are to the boundaries. A record at 512 KiB p99 with `lambda_window = 0.003` is Band A, not Band B — the guardrail obligations of Band B do not apply. Band B triggers only when at least one metric crosses its threshold.

#### Mandatory guardrails for Band B

- Retries with jitter/backoff for same-key conflicts.
- Bounded per-write mutation size.
- Reconciliation path for counter/list drift.
- Telemetry alerts for key-busy, write p95, and record growth.
- Predefined migration trigger and shard/overflow plan (see [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern).

#### 1:N consolidation safety check (explicit yes/no)

All must be answered:

- Can one root record safely hold p99 children under configured thresholds?
- Does whole-tree/full-set read locality materially benefit dominant reads?
- Is same-key write contention acceptable under `lambda_window` and load test evidence?
- Are delete/update semantics simpler and safer in consolidated form?
- Is fallback sharding trigger and migration path already defined? (See [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern.)

If any answer is "no" without an approved mitigation, do not select pure consolidation.

#### Tie-breaker rule

If multiple options remain viable after banding, choose in this order:

1. fewer cross-record writes per user operation
2. lower reconciliation complexity
3. simpler delete/cascade correctness
4. fewer sets and key formats

#### Exception rule

Any override of the rubric outcome requires:

- measured evidence,
- explicit rationale,
- migration/rollback plan,
- reviewer approval note.

### 5.3) 60-second pattern triage (optional pre-check, not final)

Use this only for fast narrowing; final decision must use 5.2.

#### Fast inputs

- `events_per_root_active_window`
- `active_window_sec`
- `contention_window_ms` (default 8)
- `burst_factor` (default 10)
- `projected_record_size_kib_p99`
- `primary_read_shape` (see glossary in 5.0)

#### Fast math

- `avg_qps = events_per_root_active_window / active_window_sec`
- `burst_qps = avg_qps * burst_factor`
- `lambda_window = burst_qps * (contention_window_ms / 1000)`

#### Fast decision

- If `lambda_window < 0.05` and size `< 1024 KiB` and read shape is not single-node-only / single-element-only: tentative consolidate.
- If `lambda_window >= 0.2` or size `>= 4096 KiB`: tentative split/shard.
- Otherwise: tentative consolidate with guardrails.

If any fast input is missing, set `TRIAGE_BLOCKED_MISSING_INPUT` and ask for missing data. Do not record a tentative pattern.

### 5.4) Decision Pack 1 — 1:N pattern selection (required)

**Prerequisite:** Complete the shared sizing gate (5.1) and contention rubric (5.2) for this relationship before entering this pack.

Choose one pattern: consolidated parent record, child-per-record with reference/index, or hybrid.

#### Required inputs

- child count per parent (`p50/p95/p99`)
- child payload size (`p50/p95/max`)
- dominant read shape (`full-set`, `subset`, `single-element` — see glossary in 5.0)
- write rate and burst profile
- lifecycle coupling (shared vs independent retention/delete)
- correctness requirement (eventual vs transactional)
- growth horizon and capacity assumptions

#### Blocker gate

If any required input is missing or low confidence:

- mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarifying questions,
- do not lock the pattern.

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| child count per parent (p50/p95/p99) | Sizing reference, traffic analysis | "What is the expected [child] count per [parent] at p50, p95, and p99?" |
| child payload size (p50/p95/max) | Sizing reference, sample data | "What is the expected [child] size distribution (p50, p95, max bytes)?" |
| dominant read shape | Access pattern analysis | "Is the primary read for [parent+children] full-set, subset, or single-element? (see glossary in 5.0)" |
| write rate and burst profile | Traffic analysis, stakeholder | "How many [child] writes per [parent] per hour at sustained and peak load?" |
| lifecycle coupling | PRD, stakeholder | "When a [parent] is deleted, are all [children] deleted too, or do they have independent lifecycle?" |
| correctness requirement | PRD, stakeholder | "Does [parent+children] require transactional consistency or is eventual consistency acceptable?" |
| growth horizon and capacity assumptions | Stakeholder, product roadmap | "Over what time horizon and at what scale should the model remain healthy?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Selection defaults

- **Consolidated:** full-set reads dominate, size stays healthy, same-key contention acceptable.
- **Split child records:** subset/single-element reads dominate, child count is high/unbounded, lifecycle diverges, or contention risk is high.
- **Hybrid:** full-set and subset/single-element reads are both first-class and projection alone is insufficient.

#### Required decision record

- selected pattern
- rejected alternatives and why
- numeric migration triggers (size, latency, contention)
- migration path (split, shard, or reconsolidate)

#### Tie-breaker

1. fewer cross-record writes on primary operations
2. lower hot-key and growth risk
3. simpler delete/cascade correctness

### 5.5) Decision Pack 2 — N:M pattern selection (required)

**Prerequisite:** Complete the shared sizing gate (5.1) and contention rubric (5.2) for this relationship before entering this pack.

Choose one pattern: dual adjacency lists, edge/association records, or hybrid.

#### Definitions

- **Edge records:** one record per relationship instance, keyed by both endpoints (e.g., `actor|target_type|target_id`), to enforce uniqueness/idempotency by record key. Each edge record costs 64 bytes of primary index.
- **Edge metadata:** data stored on the relationship itself, not on either endpoint entity — e.g., `created_at_ms`, source channel, state flags, moderation status, rank weight, audit fields. When edge metadata is minimal (e.g., timestamp-only), it can be stored inline in an adjacency list entry instead of requiring a dedicated record.

#### Required inputs

- dominant query directions (`A->B`, `B->A`, or both)
- edge metadata requirements (timestamps/state/attributes)
- uniqueness/idempotency requirements
- edge churn rate (create/delete/update)
- degree distribution and skew (`p50/p95/p99` per side)
- delete/cascade semantics and cleanup expectations

#### Blocker gate

If query directionality, uniqueness requirements, or degree skew is unknown:

- mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarifying questions,
- do not lock the pattern.

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| dominant query directions | Access pattern analysis | "Is the primary query direction [A]->[B], [B]->[A], or both equally?" |
| edge metadata requirements | PRD, stakeholder | "What metadata (timestamps, state, attributes) must be stored on each [A]-[B] edge?" |
| uniqueness/idempotency requirements | PRD, stakeholder | "Must [A]-[B] edges be unique? What happens on duplicate create attempts?" |
| edge churn rate | Traffic analysis, stakeholder | "How frequently are [A]-[B] edges created, updated, or deleted?" |
| degree distribution and skew (p50/p95/p99) | Sizing reference, traffic analysis | "What is the expected degree per [A] and per [B] at p50, p95, and p99?" |
| delete/cascade semantics | PRD, stakeholder | "When [A] or [B] is deleted, what happens to the edges? Cascade delete, orphan, or placeholder?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Selection defaults

- **Dual adjacency / list-on-entity (default when edge metadata is minimal):** both-direction list reads dominate and edge metadata is minimal (e.g., timestamp-only). Prefer this pattern when p99 relationship count fits the parent record's Goldilocks budget and PI cost of one-record-per-edge would be disproportionate. Note that `ADD_UNIQUE` on an **ordered** CDT list provides explicit uniqueness enforcement comparable to key-existence checks — "explicit uniqueness at key level" does not require edge records. The list must be ordered: on an ordered list, `ADD_UNIQUE` uses binary search for duplicate detection (O(log n)); on an unordered list, it requires a linear scan (O(n)).
- **Edge records:** edge metadata is first-class and independently mutable/queryable, edge lifecycle is independent of both entities (e.g., soft-delete, metadata updates, high churn), or edge count is unbounded and exceeds list capacity on the parent.
- **Hybrid:** low-latency list reads and rich edge metadata/correctness are both mandatory.

#### PI cost comparison (required for edge-record candidates)

Before selecting edge records, compute the PI cost differential:

- **Edge records:** `edge_count_p99 × 64 bytes` PI memory per entity pair, plus per-edge record storage overhead.
- **List-on-entity:** 1 PI entry (64 bytes) per entity; edge data stored inline in the list.

Worked example — likes at scale: at 100K likes on a content item, edge records cost `100,000 × 64 bytes = 6.4 MB` PI memory for one item's like relationships alone. List-on-parent stores all liker handles in one CDT on the content record (`~100,000 × 20 bytes = ~2 MB` payload, 1 PI entry = 64 bytes). When edge metadata is minimal (timestamp-only), the PI overhead of the edge-record approach dominates the storage cost of the actual relationship data.

When relationship degree exceeds what a single record can hold (e.g., millions of followers), the answer is sharded companion records — not edge records. Each shard holds an ordered list of reference IDs with the same structure as the non-overflowed companion. Edge records at this scale would cost `N × 64 bytes` of PI memory per relationship — at 10M followers, that is 640 MB of PI for one entity's relationships alone. See [one-to-many-relationships.md](one-to-many-relationships.md) § 5 for the companion overflow pattern.

If edge records are selected despite unfavorable PI cost, document the justification (e.g., independent edge lifecycle, rich metadata, query-by-edge requirements).

#### Required decision record

- selected pattern and key format(s)
- PI-cost comparison (edge records vs list-on-entity at p99 degree)
- why the rejected alternative is worse for this workload
- create/delete/update flow
- consistency mode (CP transactions or AP + reconciliation)
- overflow/shard trigger for high-degree entities (see [one-to-many-relationships.md](one-to-many-relationships.md) § 5 for the companion overflow pattern)
- delete/cascade cleanup strategy

#### Tie-breaker

1. correctness under retries/races
2. latency on dominant query direction
3. lower PI cost and write amplification
4. lower reconciliation burden

### 5.6) Decision Pack 3 — Hierarchical traversal shape (required)

**Prerequisite:** Complete the shared sizing gate (5.1) and contention rubric (5.2) for this relationship before entering this pack.

Choose one pattern: consolidated hierarchy per root, adjacency-per-node records, or hybrid.

#### Required inputs

- dominant traversal read (`whole-tree`, `subtree`, `single-node` — see glossary in 5.0)
- update profile (`node edits`, `subtree moves/deletes`, ranking updates)
- expected depth and breadth (`p50/p95/p99`)
- ordering semantics (time, rank, traversal-only)
- delete semantics (placeholder, hard-delete subtree, mixed)
- growth horizon and size projections per root
- contention profile on root-level mutations

#### Blocker gate

If traversal read shape, ordering semantics, or delete semantics are unclear:

- mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarifying questions,
- do not lock the pattern.

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| dominant traversal read | Access pattern analysis | "Is the primary read pattern whole-tree, subtree, or single-node for [hierarchy]? If multiple read shapes compete, what percentage of total reads does each represent? (see glossary in 5.0)" |
| update profile | Access pattern analysis, PRD | "What mutations are most common — node edits, subtree moves/deletes, or ranking updates?" |
| expected depth and breadth (p50/p95/p99) | Sizing reference, domain analysis | "What is the expected depth and breadth of [hierarchy] at p50, p95, and p99?" |
| ordering semantics | PRD, stakeholder | "How are nodes in [hierarchy] ordered — by time, rank, or traversal position only?" |
| delete semantics | PRD, stakeholder | "When a node in [hierarchy] is deleted, is it a placeholder, hard-delete subtree, or mixed?" |
| growth horizon and size projections per root | Sizing reference, stakeholder | "What is the projected size per [root] over the retention horizon?" |
| contention profile on root-level mutations | Traffic analysis, stakeholder | "How many concurrent writers can mutate the same [root] during peak load?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Selection defaults

- **Not Pack 3:** If nesting depth is strictly 1 (parent → flat children, no child-to-child references), route to Pack 1 (section 5.4) instead. Domain labels like "thread," "reply," or "comment" do not override the structural classification — the presence or absence of child-to-child nesting determines the pack.
- **Consolidated hierarchy (default):** whole-tree reads dominate, projected size remains healthy, root contention acceptable. When `lambda_window < 0.05`, `projected_record_size_kib_p99 < 1024`, and the dominant read shape is `whole-tree` or `subtree`, consolidated hierarchy is the normative default. Selecting adjacency-per-node when these conditions hold requires explicit override evidence (measured contention, subset-read dominance, or independent-child lifecycle requirement).
- **Adjacency-per-node:** single-node reads and independent node writes are the dominant access pattern, or root growth/contention is excessive. This is the correct choice when `primary_read_shape = single-node` and individual children are addressed, mutated, and queried by their own key more often than they are read as part of the root context.
- **Hybrid:** top-level reads need root locality, deeper operations need targeted node-level efficiency.

#### Single-node classification for CDT sub-element operations

`single-node` means the dominant access path is direct single-child read/write by child key — for example, repeated `get comment by id` independent of root-thread context, or frequent per-child mutations that are unrelated to root reads.

CDT sub-element operations (e.g., `map_put` on one entry inside a consolidated record, `list_append` to a nested list, `map_increment` on a counter within a map) do **not** count as `single-node` pressure by themselves. These operations execute within the existing record I/O path — the server reads the record, applies the sub-element mutation, and writes it back. No separate record lookup is required. The per-operation cost is bounded by the record size, not by the number of children.

The distinction matters: if the application likes a comment by calling `map_put` on the comment's entry within a consolidated `post_comments` record, this is a root-level record operation, not a single-node access pattern. It does not constitute evidence for splitting comments into separate records.

**When does CDT sub-element mutation become a split signal?** When the consolidated record is large enough that the per-mutation I/O cost (read + write of the full record) materially degrades write latency — i.e., the record has moved into Band B or Band C of the contention rubric in 5.2, or the `projected_record_size_kib_p99` exceeds the operational threshold. At that point, the cost argument is about record size and contention, not about the access pattern being "single-node."

#### Contrastive worked example: whole-tree vs single-node for comments

**Scenario A — whole-tree (consolidate).** A social platform where the dominant read is "load post + all comments as a thread." Comment reads always happen in root-thread context. Likes on individual comments are applied via CDT `map_increment` on the comment's entry within the consolidated record. The record stays under 512 KiB at p99. `lambda_window < 0.01`. **Classification: whole-tree. Pattern: consolidated hierarchy.**

**Scenario B — single-node (split).** A Q&A platform where the dominant read is "load one answer by answer ID" — answers are independently bookmarked, linked, and voted on without loading the parent question. Answer edits are frequent and independent. Answers have independent moderation lifecycle. **Classification: single-node. Pattern: adjacency-per-node.**

**Scenario C — the ambiguous middle (resolves to whole-tree).** A social platform identical to Scenario A, but with a secondary access path: "get comment by ID" for notification deep-links and per-comment like bursts. The root-thread read accounts for ~70% of reads; comment-by-ID accounts for ~30%. Like mutations use CDT `map_put` within the consolidated record. The record stays in Band A (`lambda_window < 0.05`, size < 1024 KiB). **Resolution: the CDT mutation path means like bursts do not create single-node pressure. The 70/30 split with Band A metrics means the default is consolidated. If the spec selects split, it must provide override evidence — e.g., measured write-p95 breach or contention exceeding Band A thresholds.**

#### Required decision record

- selected hierarchy shape
- ordering contract (where/how ordering is maintained)
- mutation paths for create/edit/delete, including subtree behavior
- split/shard triggers for size/contention
- migration path across hierarchy shapes
- read/write share between whole-tree and single-node operations (approximate percentages)
- if split is selected despite Band A metrics: override evidence and rationale

#### Tie-breaker

1. lower risk for dominant traversal and mutation operations
2. lower root-level contention and growth risk
3. simpler delete/cascade correctness

#### Ordering contract: sort-dimension pattern

**How read shape affects the role of auxiliaries.** The classified read shape determines whether auxiliary ordering structures serve as *access path infrastructure* or *optimization metadata*:

- **Consolidated hierarchy (whole-tree / subtree dominant):** A single GET returns the full record. All data — primary tree and auxiliaries — arrives in one round-trip. The auxiliaries avoid client-side recomputation (e.g., sorting 500+ comments by score) but are not separate query endpoints. If an auxiliary were dropped, the client could still derive the ordering from the primary structure at the cost of CPU and latency.

- **Adjacency-per-node (single-node dominant):** Children live in separate records. An auxiliary ordering structure on the root (e.g., a ranked list of child IDs) is the *only* way to serve "top-N by score" without scanning all child records. Here the auxiliary is load-bearing infrastructure, not optimization.

This distinction matters for sizing and failure-mode reasoning: in a consolidated design, a corrupted or missing auxiliary is recoverable from the primary structure; in a split design, it is not.

When filling in the "ordering contract" line of the decision record, identify each sort dimension the application must serve (e.g., time order, rank/score order, traversal order) and map each to a CDT structure:

1. **One sort dimension, scalar map value.** Map rank operations (`get_by_rank_range`) return entries sorted by value directly. No auxiliary structure needed. Example: `{ item_id: score }` — rank operations return entries in score order.

2. **One sort dimension, composite map value.** Structure the value as `[sort-value, other1, other2, ...]`. Map rank uses list comparison where element 0 drives rank order. No auxiliary structure needed. Example: `{ segment_id: [ttl, attrs] }` — rank operations sort by TTL. See [cdt-api.md](cdt-api.md) § "Rank with list values" for the worked example.

3. **Multiple sort dimensions.** Build one auxiliary sorted structure per additional dimension the primary structure cannot serve via rank. Maintain each auxiliary on write (add/remove):
   - **Time-ordered list:** An ordered list of IDs, append-only. Read forward = oldest first; read in reverse (`get_by_index_range` from end) = newest first. One list serves both directions.
   - **Score-ranked map:** `{ id: rank_score }`. Update via `map_put(id, new_score)` on mutation. Serve top-N via `get_by_rank_range` or sort values in application code.

4. **Skip dimensions already covered.** Trees and maps have key order; ordered lists have value order. Only dimensions not covered by the primary structure's native order require auxiliaries.

5. **One list per dimension, not per direction.** Forward and reverse reads of the same ordered dimension are served by one ordered list. Use `list_get_by_index_range(0, N)` for oldest-first and `list_get_by_index_range(-N, N)` for newest-first. Do not maintain two separate lists for the same dimension in opposite order — this doubles storage and write amplification without improving read performance (reverse-range CDT operations are O(1) for the index lookup).

**Worked example (comment ordering).** A consolidated comment tree with three display orders:

- **Traversal order:** The `tree` map (nested by parent → replies) provides structural order — no auxiliary needed.
- **Time order (oldest/newest):** `comment_order` — an ordered list of root comment IDs, append-only. Oldest first = read forward; newest first = read in reverse.
- **Rank order:** `comment_rank` — a map of `{ comment_id: like_count + repost_count }`. Updated via `map_put` on like/repost mutations. Top-N by `get_by_rank_range` or application-side sort.

Each sort dimension maps to exactly one CDT structure. No dimension requires client-side computation from raw data — the auxiliaries serve the access patterns directly. The primary structure mutation and all auxiliary updates are performed in a single `operate()` call — multiple list, map, and scalar ops within one operate execute in order, atomically and in isolation under the record lock. The primary structure and its auxiliaries are always consistent without requiring a multi-record transaction.

#### Inverse-lookup closure for consolidated hierarchies (required)

Whenever children are consolidated into a root record, check whether any lifecycle or access path requires "find all parents containing child X" — for example, user-delete cascade that must placeholder all comments by a specific user across all content.

If yes, the spec must choose exactly one supported pattern and justify it:

**(a) SI-backed parent lookup.** Maintain a `distinct_child_ids` or `distinct_authors` list bin on each consolidated root record and create a secondary index on it. On user-delete, query the SI to find all roots containing the user, then apply the cascade within each root. SI memory cost: 14 bytes per entry × number of distinct children across all parents × replication factor.

**(b) Per-child reverse index.** Maintain a per-user parent-reference structure (e.g., a `user_conversations` record listing all root IDs where the user has children). Must cover all child types (top-level and nested/reply children), not just top-level. Write amplification: one additional write per child-create to update the reverse index.

**(c) Approved async scan/reconciliation.** Scan all consolidated root records periodically or on-demand to find matching children. Requires bounded lag SLO and explicit acceptance of scan cost. Appropriate only when cascade latency requirements are relaxed (e.g., hours, not seconds).

Prohibit partial reverse indexes that only cover top-level children unless requirements are explicitly top-level-only. Include a cost/risk note (SI memory, write amplification, staleness/repair behavior) and a "why not alternatives" statement in the decision record.

### 5.7) Decision Pack 4 — Event timeline storage unit (required)

**Prerequisite:** Complete the shared sizing gate (5.1) and contention rubric (5.2) for this relationship before entering this pack.

Choose one pattern: one-event-per-record, time-bucketed records, or hybrid.

#### Required inputs

- event rate distribution (`p50/p95/p99`) for actor/recipient dimensions
- read shape (`latest page`, `deep pagination`, `filter by actor/type`)
- mutation semantics (`mark read`, delete-by-actor, expiry/TTL handling)
  - if mark-read: mark-read consumption model (`paginate-then-mark` or `server-filtered-unread`)
- retention horizon and TTL policy shape
- read/write latency targets
- expected cleanup workload (block/delete side effects)

#### Blocker gate

If event-rate distribution, read shape, or mutation semantics are unclear:

- mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarifying questions,
- do not lock the pattern.

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| event rate distribution (p50/p95/p99) | Traffic analysis, sizing reference | "What is the expected [event] rate per [actor/recipient] at p50, p95, and p99?" |
| read shape | Access pattern analysis | "Is the primary read pattern latest-page, deep-pagination, or filter-by-actor/type?" |
| mutation semantics | PRD, stakeholder | "What mutations are required — mark-read, delete-by-actor, expiry/TTL, or a combination?" |
| mark-read consumption model (if mark-read declared) | PRD, access pattern analysis | "Does the client paginate all events (with inline read/unread status) and then batch-mark displayed items as read, or does the client request only unread events from the server?" |
| retention horizon and TTL policy shape | PRD, stakeholder, compliance | "What is the retention horizon for [events]? Is TTL record-level, item-level, or both?" |
| read/write latency targets | Stakeholder, SLA | "What are the p50/p95 latency targets for [event] reads and writes?" |
| expected cleanup workload | PRD, stakeholder | "What cleanup side effects occur on block/delete (e.g., remove events from blocked user)?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Selection defaults

- **One-event-per-record:** filtering and targeted mutation/deletion are frequent, index/query cost acceptable.
- **Time-bucketed:** page reads dominate, bucket growth stays bounded, mutation semantics work efficiently in-record.
- **Hybrid:** low-latency recent reads and targeted historical filtering/mutation are both required.

#### Normative sizing rule (required before locking storage unit)

Compute the projected record size for a consolidated (single-record) timeline:

- `items_within_ttl_p99 = events_per_day_p99 × ttl_days`
- `projected_bytes_p99 = items_within_ttl_p99 × avg_event_payload_bytes`

Decision thresholds:

- If `projected_bytes_p99 > 128 KiB`: bucketed timelines are required (e.g., day buckets).
- If `projected_bytes_p99 < 64 KiB`: single-record consolidated timelines are allowed.
- If `64–128 KiB`: either pattern is acceptable with a required justification note.

If `events_per_day_p99` or `avg_event_payload_bytes` is unknown, mark `BLOCKED_MISSING_INPUT` and do not lock the storage unit.

#### Worked example — social feed and notifications

A user receives ~50 feed items/day at p95, each ~200 bytes. With 30-day retention:

- `items_within_ttl_p99 = 50 × 30 = 1,500 items`
- `projected_bytes_p99 = 1,500 × 200 = 300 KiB`

300 KiB exceeds the 128 KiB threshold, so day-bucketed storage is required. Each day bucket holds ~50 items × 200 bytes = ~10 KiB, well within the Goldilocks band. Record TTL on each bucket aligns with the 30-day retention window, enabling automatic expiry without application-level pruning.

Contrast: a low-activity notification system with ~5 notifications/day at p95, each ~150 bytes, 14-day retention: `5 × 14 × 150 = 10.5 KiB` — well under 64 KiB. A single consolidated record per user is acceptable.

#### When day buckets exceed the Goldilocks band

The normative sizing rule above determines whether day-bucketed storage is required (consolidated record too large). But day buckets themselves can exceed the Goldilocks band when per-item payloads are large. Compute the per-bucket size: `events_per_day_p95 × avg_event_payload_bytes`. If a single day bucket exceeds 128 KiB at p95, or approaches the configured `max-record-size` at max, the modeler must choose one of two responses.

**Response A — Sub-day bucketing (smaller time windows).** Split from day buckets to hour buckets (or 6-hour, 4-hour, etc.), distributing the same full-content entries across more buckets. Prefer this when: per-item payload is moderate and the event *rate* drives the oversized bucket; content is append-only with no edit/delete CRUD; and burst traffic does not concentrate enough to produce oversized hour buckets.

Trade-offs: more buckets to scan for pagination (e.g., 90 days × 24 hours = 2,160 buckets vs. 90 day buckets), higher PI cost from more bucket records, pagination must span more bucket boundaries, and content edit/delete remains a CDT mutation inside a bucket. Verify the sub-day bucket at *burst* rate, not just the average: if `burst_events_per_hour_p95 × avg_event_payload_bytes` still exceeds the Goldilocks band, sub-day bucketing alone is insufficient.

**Response B — Thin index + batch-read content records.** Day buckets hold thin ordering entries — just enough fields to reconstruct content-record keys for batch-read (e.g., `[created_at_ms, sender_handle]` at ~35 bytes). Content lives in separate one-per-record storage with per-record TTL. The bucket becomes a pagination index, not a content store. Prefer this when: per-item payload is large (hundreds of bytes to KiB); content has independent CRUD needs (edit overwrites a record, delete placeholders it, per-record TTL handles retention); or even sub-day buckets are marginal at burst rates.

Trade-offs: higher PI cost (one PI entry per content record + one per day bucket, vs. one per day bucket alone), but simpler CRUD on content records (single-record write/edit/delete vs. CDT mutation within a bucket). Per-record TTL on content aligns with retention policy without CDT pruning. The read path adds one batch-read round-trip: read day bucket → extract entries → derive content-record keys → batch-read content records. Per section 5.0.2, this is normal assembly (combining server-provided data for display), not a client-side workaround.

**Decision framework.** The choice depends on the CRUD profile and burst shape:

- If events are append-only (no edit, no individual delete, bulk expiry only) and sub-day buckets fit the Goldilocks band at burst rate → sub-day bucketing is simpler.
- If events have independent CRUD (edit, delete, per-record TTL) or per-item payload is large enough that even hour buckets are uncomfortable at burst → thin index + batch-read is cleaner.
- Tie-breaker: does the modeler prefer CDT mutations inside buckets, or single-record operations on content with an extra batch-read hop?

**Worked example — messaging channel history.** 250 msgs/day at p95, ~800 bytes per message, 120-day retention.

- *Full-content day bucket:* 250 × 800 = 200 KiB (above Goldilocks). At max (1200 msgs/day): 960 KiB per bucket, approaching risk zone with large messages.
- *Sub-day (hour) bucket:* 250 msgs / ~10 active hours = ~25 msgs/hour × 800 = ~20 KiB (in Goldilocks). But peak hour at max could spike to 250+ msgs/hour × 800 = 200 KiB. Message edit/delete requires CDT mutation inside the bucket.
- *Thin index day bucket:* 250 × 35 bytes = ~9 KiB (well within Goldilocks). Content records at ~800 bytes each with per-record TTL. Message edit = single record overwrite. Message delete = single record placeholder. At max: 1200 × 35 = 42 KiB (still comfortable).

In this case, thin index is preferred: messages have active CRUD (edit, delete) and the per-item payload is large enough that hour-bucket burst sizing is marginal.

**Classification.** This hybrid lives within Pack 4. The timeline *index* is a Pack 4 structure — subject to Pack 4 sizing, mutation-compatibility, and element-0 checks. The content records are standalone entities with standard record-per-entity design. Route each through the appropriate analysis independently.

#### TTL strategy guidance

When the event retention window aligns with a natural bucket interval (e.g., 30-day retention with daily buckets), prefer record-level TTL on buckets over item-level pruning in consolidated records. Record-level TTL has zero application-side cleanup cost and zero write amplification for expiry — the database handles removal automatically.

When using consolidated single-record timelines, expiry must be handled by application-level pruning (e.g., periodic trim of items older than the retention window). This adds write amplification proportional to the prune frequency and batch size. Document the pruning schedule, batch bounds, and expected write amplification.

#### Mutation-compatibility verification (required before locking storage unit)

Every mutation semantic declared in the required inputs above (mark-read, delete-by-actor, delete-by-type, expiry, etc.) must map to a concrete CDT operation on the chosen storage unit. If any declared mutation cannot be served by a CDT operation, the modeler must document the gap and select a fallback before finalizing the decision record. In particular, verify that the tuple element-0 choice (see below) does not place a mutation-critical field (mark-read target, dedup key) at a non-leading position where CDT list operations cannot target it selectively.

**Granularity reassessment for TTL-bounded data.** If the mutation-compatibility check reveals that multiple declared operations require client-side workarounds (set-difference logic, multi-step read-compute-write, client-side filtering) under the consolidated storage unit, apply the server-side operation preference principle (5.0.2): consider whether a simpler record granularity (e.g., one-record-per-event with an SI for listing) would serve all operations as direct server-side calls. For TTL-bounded event data, compute the PI cost of one-record-per-event: `event_rate_p99 × ttl_days × 64 bytes`. This cost is capped and self-cleaning via record-level TTL. Compare this bounded PI cost against the client-side complexity the consolidated alternative introduces. If the modeler stays with consolidation despite client-side workarounds, the decision record must document why the PI savings justify the added complexity.

**Key constraint for ordered lists of tuples:** CDT list value-based operations (`list_get_by_value`, `list_remove_by_value`, `list_get_by_value_range`, `list_remove_by_value_range`, wildcard matching) compare element-by-element from index 0. A wildcard or full-range bound at position 0 satisfies comparison before any subsequent position is evaluated. This means value-based matching only works selectively when the match field is at element position 0 (the leading element). If a mutation targets a non-leading element (e.g., "remove all tuples where element 1 = actor"), no CDT list operation can serve it as a single server-side call.

When a mutation targets a non-leading tuple element, choose and document one of these fallback strategies:

- **Read-time filtering** — Accept stale entries; filter them out at read time using client-side state (e.g., a cached block list). Zero write amplification on the mutating operation. Stale entries remain until TTL expiry. Appropriate when the retention window is short and brief staleness is acceptable.
- **Map-based storage** — Rekey the storage unit so the mutation target is a map key (e.g., `actor:timestamp`). Enables `map_remove_by_key` or `map_remove_by_key_range` for direct removal. Tradeoff: loses natural time ordering as the primary structure; read-path pagination changes.
- **Read-filter-write** — Read the record, filter client-side, write back the filtered list. General-purpose fallback. Cost: N reads + N writes across retention-window buckets. Acceptable when the bucket count is small and the operation is infrequent (e.g., block is rare relative to reads).
- **Document modeling (DB 8.1.2+)** — Restructure the storage unit as an unordered list of maps (list-of-structs) and use path expressions (`modify_by_path`, `select_by_path`) for field-level server-side filtering and mutation. Eliminates the positional constraint entirely — any field is addressable by name. This resolves mutation-compatibility gaps where multiple non-leading fields require server-side operations (mark-read, block-actor, type-filtered query) that baseline CDT patterns cannot serve. DB 8.1.2 is the production prerequisite for path expressions (8.1.1 was preview). Version-gated; requires a baseline CDT fallback for environments below DB 8.1.2. See [Document modeling with path expressions in path-expressions.md](path-expressions.md#document-modeling-with-path-expressions).

**Worked example — block side effect on day-bucketed notifications:**

Notification tuples are `[created_at_ms, actor, notif_type, ...]` in an ordered list. The block operation requires removing all tuples where element 1 = blocked actor. Element 0 is `created_at_ms`, so value-based CDT operations cannot selectively match on element 1. With 7-day TTL and daily buckets (up to 7 buckets), the options are: (a) read-time filtering using the client's cached block list (zero writes, stale for up to 7 days), (b) read-filter-write across 7 buckets (7 reads + 7 writes, but block is infrequent). If the notification read path already filters by a block list for other reasons (e.g., feed items), option (a) is consistent and preferred.

#### Tuple element-0 choice (mandatory gate — required before locking tuple structure)

Before locking the tuple or map-value structure for an event timeline, answer both questions and document the answers in the decision record:

1. **Time-precision question:** Does the display requirement need sub-bucket time precision (strict interleaved ordering of events from different sources within a single day bucket), or is approximate recency sufficient (day-level ordering from bucket keys + insertion order within a bucket)?
2. **Identity question:** Does the event type have a natural identity — a combination of fields that uniquely identifies the logical event (e.g., `[actor, event_type, target_type, target_id]` for a notification, or `message_id` for a per-message notification)?

**Resolution:**

- **Approximate recency sufficient AND natural identity exists → identity-first is the normative default.** Place identity fields at the leading tuple positions (or use them as map keys). With `LIST_ORDERED, ADD_UNIQUE, NO_FAIL`, duplicate writes for the same event are automatically and idempotently deduplicated. Append order within a bucket provides approximate recency; day-bucket keys provide day-level ordering. Together these satisfy "recent first" pagination without a timestamp at element 0. Choosing timestamp-first when these conditions hold requires explicit override justification in the decision record.
- **Strict time ordering required → timestamp-first is correct.** When the display requirement is strict time ordering within a page (e.g., a content feed where items from different authors must interleave by exact publication time), timestamp at element 0 is correct. Dedup in this case relies on the event's natural uniqueness (distinct content IDs from distinct authors) rather than `ADD_UNIQUE`. Document why sub-bucket time precision is needed.

**Why this is a gate, not advisory guidance.** Placing a timestamp at element 0 has a silent cost: it defeats `ADD_UNIQUE` deduplication. When the same logical event can arrive more than once with different timestamps (retries, concurrent writers, at-least-once delivery), tuples that differ only in element 0 are treated as distinct values by `ADD_UNIQUE`. Additionally, timestamp-first ordering places mutation-critical fields (mark-read flags, dedup keys) at non-leading positions, where CDT list operations cannot target them selectively — a conflict that the mutation-compatibility verification (above) is designed to catch. Answering the two gate questions before choosing the structure prevents the modeler from defaulting to the feed pattern for event types where it is inappropriate.

**Cross-check:** After locking the element-0 choice, verify it against the mutation-compatibility verification above. If the chosen structure places a mark-read target or dedup-critical field at a non-leading position, the mutation-compatibility check will surface the conflict and require a fallback or redesign. **Document modeling resolution (DB 8.1.2+):** If the target environment runs Aerospike 8.1.2 or later, the document modeling pattern — list-of-structs (unordered list of maps with path expressions) — eliminates the positional problem entirely. Fields are addressed by name, and `modify_by_path` targets any field directly regardless of position. This avoids the element-0 gate trade-off for entities with both identity-based dedup and field-level mutation requirements. See [Document modeling with path expressions in path-expressions.md](path-expressions.md#document-modeling-with-path-expressions).

#### Map-keyed storage variant

When the dominant mutations are key-targeted (mark-read by event ID, dedup by natural key, delete by actor+target), a map keyed by the event's natural identity can be more ergonomic than an ordered list of tuples.

**Structure:** key = natural identity (e.g., `message_id` for notifications), value = tuple of remaining fields (`[type, read, created_ms, actor, ...]`).

**Trade-offs vs ordered list:**

| Dimension | Map-keyed | Ordered list of tuples |
|-----------|-----------|----------------------|
| Key-targeted mutation | O(log N) via `map_put` / `map_remove_by_key` | O(N) scan or requires element-0 = identity to use `list_get_by_value_interval` |
| Time-ordered pagination | `map_get_by_rank_range` when the value's leading element is the timestamp (rank = time order, no auxiliary needed); otherwise requires parallel order list or client-side sort | Native via `list_get_by_index_range` |
| Dedup on write | Implicit — map keys are unique | Requires `ADD_UNIQUE` + identity at element-0 |
| Storage overhead | Map key + value per entry; slightly larger than equivalent tuple | Compact ordered tuples |

**When appropriate:** (a) the element-0 gate selects identity-first, AND (b) key-targeted mutations require individual-event server-side operations beyond post-display batch updates — specifically: write-path dedup under at-least-once delivery, server-filtered-unread queries, or frequent per-event status toggles (typing indicators, presence), AND (c) time-ordered pagination can tolerate `map_get_by_rank_range` on the value's leading timestamp element, client-side sort, or a parallel order list.

**When NOT appropriate:** (a) strict server-side time ordering is the dominant read pattern and mutations are rare or batch-only, OR (b) the mark-read consumption model is paginate-then-batch-mark — the client already holds the event IDs from the read response and mark-read is a post-display fire-and-forget operation. In both cases, the identity-first ordered list of tuples serves mark-read via `list_get_by_value_interval` on the leading identity or via a bounded read-filter-write on small day buckets, and remains the normative default.

**Worked example — Twitter notification mark-read: consumption model determines structure.**

Scenario: a user's notification timeline (likes, retweets, follows, replies). Day-bucketed, ~50 events/day at p95, identity-first element-0 (natural key = `[actor, notif_type, target_id]`).

- **Paginate-then-mark model.** Client calls `GET /notifications?page=1`. Server reads today's bucket via `list_get_by_index_range(-20, 20)`, returning tuples with inline `read` flags. Client renders the page, then calls `POST /notifications/mark-read` with the IDs it just displayed. Server writes those flags back. On a ~50-element bucket, even a read-filter-write is trivially fast. The ordered list serves both operations with native CDT calls. The map-keyed variant adds complexity (rank-based pagination or parallel order structure) with no measurable latency benefit at this bucket size. **Verdict: ordered list is the normative default.**
- **Server-filtered-unread model.** Client calls `GET /notifications?unread=true`. Server must return only events where `read = false`, filtering within the bucket. With an ordered list, `read` is a non-leading tuple field — no CDT list operation can filter on it selectively. The server must read the full bucket and filter application-side, or maintain an auxiliary unread index. With a map-keyed structure, this is still not free (map values cannot be filtered by a non-key field either), but the map enables a companion structure: a separate `unread_ids` ordered list trimmed on mark-read, with the map serving as the content store for O(log N) key lookups. **Verdict for baseline CDT (all versions):** map-keyed with companion `unread_ids` list is a candidate; evaluate against one-event-per-record with an expression filter as an alternative. **On Aerospike 8.1.2+ (document modeling):** the document modeling pattern — list-of-structs (unordered list of maps with path expressions) — resolves this directly. `select_by_path` with a filter on `read == false` returns only unread elements in a single server-side operation, and `modify_by_path` handles mark-read, block-actor, and type-filtered queries with the same mechanism. DB 8.1.2 also adds `mapKeysIn` and `andFilter` context types for efficient key-set selection within path expressions. See [Document modeling with path expressions in path-expressions.md](path-expressions.md#document-modeling-with-path-expressions) for the full pattern with experimentally validated operations.

The consumption model is the fork. A modeler that sees "mark-read required" but does not ask **how** the client consumes unread state will over-index on the map variant.

#### Required decision record

- selected storage unit and key format
- sizing math (items_within_ttl_p99, projected_bytes_p99, threshold comparison)
- tuple element-0 choice (timestamp-first or identity-first) with rationale
- pagination contract
- mark-read and delete-side-effect mutation path
- mutation-compatibility verification (every declared mutation maps to a concrete CDT operation or a documented fallback)
- TTL/expiry contract (record-level, item-level, or both) with operational tradeoffs
- bucket rollover/split thresholds and migration path

#### Tie-breaker

1. latency and correctness on dominant read/update operations
2. lower operational cleanup complexity
3. lower risk of hot buckets or oversized records
4. natural TTL alignment (record-level TTL preferred over application-level pruning)

### 5.8) Decision Pack 5 — Deferred cleanup strategy (required for multi-record delete/update side effects)

**Prerequisite:** This pack is used alongside a pattern pack (5.4–5.7), not instead of one. Complete the pattern pack first; then use this pack for the cleanup/cascade side effects.

Choose one strategy: synchronous full cleanup, deferred cleanup with scheduled runner, or hybrid.

#### Required inputs

- side-effect fan-out per operation (`p50/p95/p99` records touched)
- max acceptable user-facing latency for the initiating operation (`p95` target)
- consistency requirement for side effects (`immediate` vs `eventual within X`)
- acceptable reconciliation lag (`minutes/hours`)
- idempotency posture for each cleanup mutation (safe retry yes/no)
- failure/retry policy requirements (max attempts, backoff, dead-letter/escalation)
- read behavior during pending cleanup (`hidden`, `placeholder`, `mixed`)
- operational trigger model (scheduler cadence, manual endpoint, event-triggered)
- observability requirements (backlog, oldest-pending age, success/failure rate)

#### Blocker gate

If side-effect fan-out, consistency requirement, reconciliation lag, or pending-read behavior is unclear:

- mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarifying questions,
- do not lock synchronous vs deferred strategy.

#### Clarification mapping (use when input is missing)

| Required input | Typical source | Question template if missing |
|---|---|---|
| side-effect fan-out (`p50/p95/p99`) | Access pattern matrix, sizing worksheet | "For [operation], how many related records are typically touched at p50, p95, and p99?" |
| initiating-operation latency target (`p95`) | SLA/SLO, stakeholder | "What p95 latency must [operation] meet at user-facing API level?" |
| side-effect consistency requirement | PRD, stakeholder | "Must side effects be complete before response, or can they converge within a bounded delay?" |
| acceptable reconciliation lag | Product/ops stakeholder | "What is the maximum allowed delay before all cleanup side effects are complete?" |
| idempotency posture | Engineering design | "Are all cleanup mutations safe to retry without double-apply effects?" |
| failure/retry policy | Ops/engineering | "What retry/backoff and escalation policy is required for repeated cleanup failures?" |
| read behavior during pending cleanup | Product requirements | "While cleanup is pending, should entities be hidden, shown as placeholders, or mixed by surface?" |
| operational trigger model | Runtime ops | "Should cleanup run hourly/daily, on-demand via endpoint, or event-triggered?" |
| observability requirements | SRE/ops | "Which cleanup metrics and alerts are mandatory?" |

If the source column says the input should come from a document that exists but doesn't contain the answer, mark `MISSING` and ask.

#### Selection defaults

- **Synchronous full cleanup:** low fan-out and immediate consistency required.
- **Deferred cleanup:** high fan-out and eventual consistency acceptable within bounded lag.
- **Hybrid:** immediate critical invariants + deferred non-critical side effects.

#### Required decision record

- selected cleanup strategy (`sync` / `deferred` / `hybrid`)
- why rejected alternatives are worse for this workload
- **job-discovery mechanism:** one of (a) dedicated `cleanup_jobs` set with state machine, (b) SI/query on entity `status` bin, or (c) external queue. Include brief justification for the choice. Dedicated set is recommended when cascade fan-out is large and resumable progress tracking is needed; SI on status is simpler but has O(N) discovery cost; external queue is appropriate when queue infrastructure already exists.
- **sync-to-async promotion thresholds:** numeric values for when a synchronous cleanup path should hand off to async — e.g., `records_touched_p95` (number of records), `elapsed_ms_p95` (wall-clock time), `retry_count` (consecutive failures). If any threshold is exceeded during a synchronous attempt, the operation enqueues the remainder for async processing.
- lifecycle state contract (for example `status=deleting/deleted`, `cleanup_state=pending/running/done/failed`)
- trigger contract (for example internal endpoint called hourly)
- batch bounds and continuation/cursor policy
- retry/backoff/escalation policy
- read-path behavior while cleanup is pending
- reconciliation SLO and alert thresholds (including `backlog_age_slo` — maximum age of the oldest pending cleanup job)
- rollback/disable plan if runner causes operational regressions
- **per-operation discovery structure** (required for each deferred cleanup operation): How does the cleanup process find the records to modify? For each deferred cleanup path, document:
  - **Discovery structure:** The specific set, key, index, or companion record that enables enumeration of affected records. If no efficient discovery path exists, either add one (companion record, SI, or reverse index) or explicitly document that the cleanup accepts scan cost with a bounded-lag SLO.
  - **Normal-operation cost:** One-sentence write amplification or memory cost of maintaining the discovery structure during normal (non-cleanup) operations.
  - If a deferred cleanup path is declared without a concrete discovery structure, mark `BLOCKED_MISSING_INPUT` and do not finalize the cleanup strategy.

**Worked example — user-delete like cleanup.** A deferred cleanup operation must remove or anonymize likes placed by a deleted user across all content. Without a `user_likes` companion record (or SI on a `liker_ids` bin), the only discovery path is a full namespace scan — O(N) on total content records, not bounded by the user's activity. **Resolution:** Add a `user_likes` companion record (key = `{handle}`, ordered list of `content_id` values the user has liked). On like, append to `user_likes`; on cleanup, iterate the list to find and modify each content record's like structure. Normal-operation cost: one additional write per like event to maintain the companion.

#### Discovery patterns for time-bucketed data

When the cleanup target is time-bucketed records (Pack 4 day buckets), the discovery structure must enumerate which buckets exist. Three patterns:

| Pattern | Mechanism | Normal-operation cost | Best when |
|---------|-----------|----------------------|-----------|
| **1. Activity-day list on parent** | Parent record (e.g., channel) maintains an ordered list of `YYYY-MM-DD` strings for days with activity. Append on write via `ADD_UNIQUE`. Cleanup iterates this list to construct bucket keys. | One list append per active day (deduplicated via `ADD_UNIQUE`). | Retention is long or unbounded. Avoids scanning years of date keys. Parent record already exists and the day list stays small relative to other bins. |
| **2. Date-range iteration from metadata** | Compute date range from parent creation date to now (or deletion date). Iterate day keys deterministically. | Zero — no companion structure. | Retention is short and bounded (e.g., 90-day window). Overhead of attempting reads on non-existent bucket keys (`KEY_NOT_FOUND` returns cheaply) is tolerable. |
| **3. SI on parent-id bin within bucket set** | Secondary index on the parent identifier bin across all bucket records. Query returns all buckets for the parent. | SI memory for the indexed bin. | SI already exists for other access patterns. Adding a purpose-built SI solely for cleanup is usually not justified. |

**Decision heuristic:** Use pattern 1 when retention is long or unbounded; use pattern 2 when retention is short and bounded; use pattern 3 only if the SI already exists for other purposes. Document the selected pattern and its normal-operation cost in the decision record.

#### Tie-breaker

1. correctness on required invariants
2. latency compliance on initiating operation
3. lower operational risk (retry safety, observability, bounded backlog)

#### Example pattern (generic cleanup shape)

- Initiating operation sets lifecycle status and `cleanup_state=pending`.
- Reads immediately hide or placeholder entities per contract.
- State machine for cleanup jobs: `pending → running → done | failed`. A job stays in `running` only while a worker holds it; on worker crash or timeout, the job reverts to `pending` for retry.
- Internal cleanup endpoint is invoked on schedule (for example, hourly) and processes bounded batches.
- Worker applies idempotent cleanup operations, advances cursor, and marks `done` when complete.
- If retries exceed threshold, mark `failed` and raise alert.
- Monitor `backlog_age_slo` (maximum age of oldest pending job) and alert when the oldest pending job exceeds the reconciliation SLO.

#### Deleted-entity display pattern (zero write amplification)

When a deleted entity's identifier is denormalized into many records (sender handle on thousands of messages, author handle on comments, actor handle in reaction lists), the delete cascade must change how those references display. The naive approach — scan and update every referencing record — produces write amplification proportional to the entity's lifetime activity. This pattern eliminates that cost.

**The pattern.** Create a small dedicated lookup set (for example, `deleted_users`). Key = the deleted entity's ID. Bins = metadata (`deleted_at_ms`, optionally the entity type if the set covers multiple entity kinds). On delete, one write adds the entity to the lookup set. The read path checks this set when rendering references and substitutes a placeholder (e.g., "[Deleted user]"). Referencing records are never modified for display purposes.

**When to use it.** The fan-out of referencing records is large (hundreds to tens of thousands), those records have their own lifecycle or TTL, and the display change is cosmetic (replacing a name or label) rather than structural (removing a data dependency). This is one concrete implementation of the `read behavior = placeholder` option in the Pack 5 required inputs above.

**When NOT to use it.** When the referencing record count is small (bounded, low fan-out) and direct updates can run within the sync or async latency budget, updating the referencing records is simpler and avoids read-path indirection. Also not appropriate when the delete must structurally remove data from referencing records (e.g., removing a user's entries from reaction lists) — that still requires scan-based cleanup regardless of the display pattern.

**Read-path integration.** The lookup set can be checked per-render (O(1) GET by key), batch-loaded on session start (batch-read a list of recently encountered handles), or cached client-side with a short TTL. The choice depends on the expected size of the lookup set and the read frequency. For most applications, the set stays small (deleted entities accumulate slowly) and a session-start batch-load or short-lived client cache is sufficient.

**TTL on the lookup set.** If all referencing records have bounded retention (e.g., 120-day message TTL), set the lookup entry's TTL to `max_retention + buffer` — the entry auto-expires once all stale references are gone. If referencing records are permanent, the lookup entry must also be permanent (or periodically confirmed still needed).

**Worked example — user delete in a messaging app.** A user has sent 8,000 messages across 200 channels over 120 days. Naive approach: scan the `messages` set for this sender and update each record — 8,000 writes, bounded by retention but expensive. Lookup-set approach: one write to `deleted_users` (key = handle, bins = `{deleted_at_ms}`). Client display logic: when rendering a message sender, check the `deleted_users` set (GET or cache hit). If the handle is present, display "[Deleted user]." Messages auto-expire via their own 120-day TTL. The lookup entry can expire at 120 days + buffer, after which no referencing records remain.

### 5.8.1) Derived-metric patterns: counter-snapshot (cross-cutting — apply after packs)

Some access patterns require a metric that spans two records owned by different actors — for example, "how many messages since this user last looked at this channel." This is not a relationship-shape decision (Packs 1–4) or a cleanup strategy (Pack 5). It is a cross-entity derived metric: the answer lives on neither entity alone but is computed from a value on each.

The counter-snapshot pattern addresses this class of problem:

- A **producer entity** (e.g., channel, feed source, forum thread) maintains a monotonic counter, incremented on each qualifying event (message send, item publish, post creation). The counter is never decremented.
- An **observer entity** (e.g., user workspace state, follower profile) stores a snapshot of the producer's counter value at the time the observer last consumed the data (last channel view, last feed visit).
- The derived metric = `producer.counter − observer.snapshot`. Computed at query time from two reads. No materialization fan-out on the write path.

#### When to apply

- An access pattern requires "how many X since observer last looked" or "what's new since last visit."
- The naive alternatives fail: fan-out counter (increment a per-observer counter on every event) has write amplification proportional to observer count; count-on-read scan (count items between cursor and head) exceeds latency SLO when the gap is large or data is bucketed.
- The metric is approximate: the monotonic counter never decrements on event deletion, so the derived count may over-count by the number of deletions since the observer's snapshot. This is acceptable when the product tolerates small over-counts (standard for messaging unread, notification badges, forum "new posts" indicators).

#### Required inputs (before locking the pattern)

| Input | Description | Question if missing |
|-------|-------------|---------------------|
| `observer_count_p99` | How many observers per producer (e.g., channel members) | "How many users observe the same [producer] at p99?" |
| `event_rate_p99` | Events per producer per day (e.g., messages per channel per day) | "How many [events] per [producer] per day at p99?" |
| `derived_metric_read_slo` | Latency target for the metric query (per-producer and batch) | "What is the required p95 latency for computing [metric] per [producer] and across all [producers] for one observer?" |
| `batch_metric_read_shape` | Single-producer query vs batch across all producers for one observer | "Is the primary read single-[producer] or batch across all [producers] the observer belongs to?" |
| `delete_impact_posture` | Over-count acceptable, or exact count required | "When a [event] is deleted, is it acceptable for the [metric] to temporarily over-count until the observer's next visit?" |

#### Blocker gate

If `delete_impact_posture` is "exact count required," the counter-snapshot pattern does not apply — the monotonic counter cannot serve exact counts across deletions. Document the alternative: either a fan-out counter with decrement (accepts write amplification) or a count-on-read scan with caching (accepts read latency). Mark `BLOCKED_DESIGN_MISMATCH` and select a different approach.

If `derived_metric_read_slo` or `batch_metric_read_shape` is unknown, mark `BLOCKED_MISSING_INPUT` and ask.

#### Worked example — messaging unread counts

A messaging application requires per-channel unread counts (< 20ms p95) and batch unread for all channels in the sidebar (< 100ms p95). Channels have up to 400 members at p99. The busiest channels see ~150 messages/day.

**Producer:** Channel record (already designed in Pack 1 or Pack 4). Add bin `msg_cnt` (int64, monotonic). Incremented on every message send:

```
operate(channel_key, [increment("msg_cnt", 1)])
```

One write to the channel record. No fan-out to observers.

**Observer:** User workspace state record (one per user per workspace). Add `cursors` map bin (`K-ordered map, {channel_id: msg_cnt_at_read}`). Updated when the user views a channel:

```
operate(observer_state_key, [map_put("cursors", channel_id, current_channel_msg_cnt)])
```

One write to the user's own record. Single-writer (only the user updates their own cursors).

**Unread computation:**

| Operation | Path | Records read | Payload |
|-----------|------|:------------:|---------|
| Per-channel unread | Read `observer_state` → `cursors[channel_id]`. Read `channel` with bin projection `[msg_cnt]`. Subtract. | 2 | ~8 bytes from channel |
| Batch unread (sidebar) | Read `observer_state` → full `cursors` map + `channels` list. Batch-read all channel records with bin projection `[msg_cnt]`. Subtract per channel. | 1 + N | At p95 (80 channels): ~1 KiB total |

Both paths are well within the stated SLOs.

**Write cost summary:**

| Event | Writes | Target record | Fan-out |
|-------|:------:|---------------|---------|
| Message send | 1 | Channel (`msg_cnt` increment) | None |
| Channel view | 1 | User's `observer_state` (`cursors` map put) | None |

**Approximation trade-off:** `msg_cnt` is monotonic — it never decrements when a message is deleted. If 3 messages are deleted between the observer's last snapshot and now, the unread count over-reports by 3. The over-count self-corrects on the observer's next channel view (the snapshot advances past the deletions). This is standard behavior for messaging applications and matches typical product expectations.

**Block/filter interaction.** When the observer filters events at read time (e.g., hiding messages from blocked users), the counter-snapshot metric will over-count by the number of filtered events between the snapshot and now. This is the same class of approximation as the delete over-count: the monotonic counter does not decrement for filtered events. Three options:

| Option | Mechanism | Trade-off |
|--------|-----------|-----------|
| **(a) Accept the over-count** | No additional writes or reads; counter remains monotonic | Over-count self-corrects on next view (snapshot advances past filtered events). Consistent with delete over-count behavior. |
| **(b) Per-observer decrement on block** | When observer blocks a user, write a decrement equal to that user's messages between snapshot and now | Write amplification proportional to blocked user's message volume in each affected channel. Requires scanning bucket records at block time. |
| **(c) Subtract at read time** | Maintain a per-channel per-blocked-user message counter; subtract from the unread delta at read time | One additional counter increment per message send (keyed by `channel:author`). Read path adds one `map_get_by_key_list` for the observer's blocked set. |

**Normative default:** option (a) unless the product requires exact filtered counts. Options (b) and (c) should be evaluated only when the product explicitly rejects over-counting for blocked-user messages and the expected block-to-message ratio justifies the write or read amplification.

**Rejected alternatives:**

| Alternative | Mechanism | Why rejected |
|-------------|-----------|-------------|
| Fan-out counter | On every message send, increment a per-user unread counter for every channel member | Write amplification = `member_count` per message. At 400 members and 150 msgs/day: 60,000 counter writes/day/channel. |
| Count-on-read scan | On each unread query, count messages between cursor timestamp and channel head across day buckets | For a user who hasn't checked a channel in 7 days with daily buckets: 7 bucket reads + element counting. Exceeds 20ms p95 SLO for inactive channels. |
| Separate counter record | Dedicated `unread_counter` record per user per channel, incremented on send, reset on view | One additional record per user-channel pair. PI cost: `users × channels × 64 bytes`. At 8K users × 80 channels: ~40 MB PI for counter records alone. The `cursors` map on `observer_state` achieves the same result with zero additional PI. |

#### Generalization

The counter-snapshot pattern applies wherever an observer needs "count of events since I last looked" across a shared resource:

- **Notification badges:** producer = notification source, counter = notification count, observer = user's badge snapshot.
- **Forum unread threads:** producer = forum/category, counter = thread count, observer = user's last-visit snapshot.
- **Feed "new since last visit":** producer = feed source, counter = item count, observer = follower's last-seen snapshot.

The structural requirement is always the same: (a) monotonic counter on the producer, (b) snapshot on the observer, (c) difference at read time. The pattern trades exact accuracy on delete for zero write-path fan-out.

#### Required decision record

When applying the counter-snapshot pattern, document:

- producer record, set, and counter bin name
- observer record, set, and snapshot bin/structure
- per-producer read path (operations, records, latency estimate)
- batch read path (operations, records, latency estimate)
- producer write path (increment trigger and operation)
- observer write path (snapshot update trigger and operation)
- approximation trade-off (what causes over/under-count, self-correction mechanism, product acceptance)
- rejected alternatives with rationale and reconsider triggers

### 5.9) Developer walkthrough checkpoint (per entity group)

After drafting the schemas, relationship decisions, and access patterns for a group of related entities and their relationships, pause and trace concrete developer scenarios through the schema before moving to the next group.

**When to apply:** After completing the decision packs for a group of entities that share relationships. An entity in isolation is rarely meaningful — the interesting decisions (consolidation vs split, companion records, fan-out, cascade) live at relationship boundaries. Natural entity groups for a social application might be: "Post + comments + likes", "User + follows + feed + notifications", "User + blocks + notification cleanup".

**What to do:** For each entity group, trace 2–3 scenarios end-to-end through the drafted schema:

1. **Create-and-read:** A write followed by the most common read that consumes it. Example: "Author publishes a post → follower opens feed → sees the post." Trace each step to a specific record key, CDT operation, and any batch reads.

2. **Mutation:** An update or interaction that touches multiple records. Example: "User likes a comment → which records are written, what does the client need to know to issue each write?" Verify the client has all required context (record key, bin path, element identity) from information already available in the workflow — not from an additional lookup.

3. **Cleanup or cascade:** A delete or side-effect operation that spans multiple records. Example: "Author deletes the post → what cascade steps, which records, in what order?" Verify each step is mechanically clear from the schema.

**For each scenario, verify:**

- Every operation maps to a specific record key and CDT operation — no ambiguity about which record to target or what the client sends.
- Any information the client needs (e.g., which day bucket a notification lives in for mark-read) is already available from a prior step in the same workflow, not a new lookup.
- If an operation feels unclear, distinguish:
  - **(a) Schema gap:** the design is actually ambiguous or incomplete — a developer could not implement it without guessing. This requires revision before continuing.
  - **(b) Normal engineering:** the design is clear but combining two well-defined patterns requires thought. This is expected and is not a design gap.

**Calibration rule:** If the walkthrough reveals only (b)-type friction, the schema is sound. Do not flag normal engineering work as a guidance gap or a design issue. Only (a)-type findings require resolution before moving to the next entity group.

**Gate:** Do not move to the next entity group until the walkthrough passes. If a walkthrough reveals a schema problem (type a), revise the schema for this group before continuing.

### 5.9.2) Implementer review checkpoint (per entity group, mandatory)

After the developer walkthrough (5.9) passes, submit the group's schemas, bin tables, and walkthroughs for review by someone with an implementer's perspective — not the architect who designed them. The architect's walkthrough catches structural gaps; this checkpoint catches spec-clarity gaps that the architect cannot see because they already know what they meant.

**Reviewer:** A team member who will implement against the spec (or, in LLM workflows, a sub-agent loaded with the implementer persona and instructed to review the spec as a consumer). The reviewer must not have participated in the design decisions for this group.

**Scope:** The current group's schemas (sets, keys, bin tables), developer walkthroughs, any cross-group interfaces already defined, and the PRD (or the relevant PRD sections for this group). The reviewer evaluates two things: (1) whether the spec is clear enough to implement from, and (2) whether the data model enables the PRD requirements to be implemented — i.e., "can I build what the PRD asks for using this schema?" The reviewer does not evaluate architectural decisions (sizing, contention, pattern selection).

**Review categories:** The reviewer evaluates the spec against five categories:

1. **Ambiguities** — Places where two reasonable implementers might interpret the spec differently and produce incompatible code. Highest priority.
2. **Missing operation details** — Places where the spec describes what should happen but does not provide enough detail for the implementer to write the specific Aerospike `operate()` call without guessing (e.g., CDT context paths for nested map operations, specific list/map policies, exact operation order in a multi-op `operate()`).
3. **Cross-record coordination gaps** — Multi-record operations where the spec does not specify expected behavior on partial failure (e.g., step 2 succeeds, step 3 fails). An implementer following TDD needs to know what the expected state is.
4. **Unclear key derivation** — Places where constructing a record key requires information the implementer might not have at the call site, or where the key format description is ambiguous (e.g., composite keys with delimiters that conflict with value characters, unspecified string encoding for hash inputs).
5. **Requirements implementability gaps** — Places where a PRD requirement cannot be fulfilled (or cannot be fulfilled within stated SLOs) using the provided schema, or where the mapping from a specific requirement to the data operations needed to satisfy it is unclear.

**Output:** A selective list of findings (5–15 items across the group). Each finding is 2–3 sentences and must include both: (a) a spec reference — where in the data model the issue appears, and (b) a requirements reference — which PRD requirement, access pattern, or stated invariant the finding is anchored to. Findings that lack a requirements anchor are out of scope and must not be submitted.

**What the reviewer should NOT do:**

- Suggest redesigns or alternative patterns.
- Comment on sizing math, contention analysis, or architectural decisions.
- Raise concerns about things clearly deferred to later groups or later spec sections.
- Flag normal implementation decisions (e.g., choice of CDT list removal primitive when the spec says "remove") as ambiguities.
- Raise findings that are not anchored to a specific PRD requirement, access pattern, or stated invariant. Opinions, preferences, or speculative concerns without requirements-based reasoning are out of scope.

**Architect triage (mandatory):** The architect classifies each finding as one of:

- **Fix in spec** — The finding identifies a genuine clarity gap. Fix before presenting to the stakeholder, or explicitly defer with rationale.
- **Normal implementation decision** — The finding is within the implementer's expected engineering judgment. Document the rationale for not adding it to the spec.
- **Rejected — no requirements anchor** — The finding does not cite a specific PRD requirement or access pattern, or the cited requirement does not support the concern. The architect documents why the anchor is missing or invalid and discards the finding.

**Gate:** All findings must be triaged before proceeding to the stakeholder review (5.9.3). Findings classified as "fix in spec" must be either resolved or explicitly deferred with rationale documented in the stakeholder review summary.

### 5.9.3) Stakeholder review checkpoint (per entity group)

After the developer walkthrough (5.9) and implementer review (5.9.2) pass for an entity group, present a brief decision summary to the stakeholder before proceeding to the next group. This checkpoint surfaces silent assumptions and judgment calls that deterministic gates do not resolve.

**Required contents of the decision summary:**

- **Schemas:** Sets, keys, and bin structure for this group. Enough detail for the stakeholder to understand what records exist and how they are accessed — not the full spec, but the key design choices.
- **Assumptions log:** Every judgment call where the modeler chose between alternatives not fully resolved by deterministic gates. For each: what was chosen, what alternatives existed, and why this one. Examples: which author handle enters the ID recipe for multi-author posts; notification payload structure for deep-link navigation; whether retweets create association entries or new content records; cleanup thresholds chosen without stakeholder input.
- **Walkthrough results:** The 2–3 scenarios traced in 5.9 and whether they passed cleanly. If any (b)-type friction was noted (normal engineering, not a schema gap), mention it briefly for context.
- **Implementer review results:** The findings from 5.9.2 and the architect's triage. For each finding: what was raised, how it was classified, and (for "fix in spec" items) whether it was resolved or deferred.
- **Open items:** Anything deferred, flagged as uncertain, or dependent on a later entity group.

**Stakeholder response options:**

- **Approve** — Modeler proceeds to the next entity group.
- **Request revision** — Modeler revises the schema for this group and re-runs the developer walkthrough (5.9), implementer review (5.9.2), and this checkpoint before proceeding.
- **Ask follow-up questions** — Modeler answers, updates the schema if needed, and re-presents.
- **Defer review** — Stakeholder explicitly accepts the current state without detailed review. The modeler proceeds but logs the deferral.

**Gate:** Do not proceed to the next entity group until the stakeholder approves or explicitly defers review. Update the entity-group plan status for this group: `done`, `blocked`, or `revision needed`.

---

## 6) Index strategy and memory rationale

For each index, require:

- Query it serves.
- Why key+batch is insufficient.
- Expected indexed population and selectivity.
- Capacity estimate (entry count, overhead, replication effect).
- Deletion/TTL behavior expectations.

Defaults:

- Do not add SI "just in case."
- Prefer direct key access and bounded batch reads for primary traffic.
- Use expression indexes (DB 8.1+) for selective/computed indexing. Expression indexes can use `cond(..., unknown())` to create sparse indexes (only records meeting the condition are indexed), reducing memory and improving query cost. With path expressions (DB 8.1.2+), expression indexes can index values extracted from nested CDT structures (e.g., `CdtExp.selectByPath` to index a field across all elements of a list-of-maps). See [expressions.md](expressions.md#secondary-index-expressions-expression-indexes) for expression index patterns.

---

## 7) Version gates and fallbacks (mandatory)

Before using advanced features, lock DB and client versions and define a fallback.

| Feature | Minimum version | Assumptions | Fallback if unavailable |
|--------|------------------|-------------|-------------------------|
| **Multi-record transactions** | Aerospike DB 8+ (strong-consistency namespace) | Atomic multi-record updates required for relationship integrity. | AP flow with idempotent writes + verify-after-write + background reconciliation. |
| **Expression indexes** | Aerospike DB 8.1+ | Need sparse or computed-value index. | Persist derived value in a regular bin and index that bin, or redesign to key+batch. |
| **Path expressions** | Aerospike DB 8.1.2+ (production). 8.1.1 was preview; 8.1.2 adds `mapKeysIn`, `andFilter` context types and is the documented production prerequisite. | Nested list/map filtering or indexing in-place. | Denormalize selected fields into dedicated bins/records; use CDT context + classic expressions where possible. |

Also verify client support against the client matrix for your language/runtime.

**Features-not-used declaration (required).** The version gates table above covers features the model relies on. Additionally, the spec must include a features-not-used declaration for each advanced feature listed in the table that the model does NOT use. For each, state: the feature name, `used: no`, and a one-line `reason_not_used` — for example, "Path expressions: used: no. Reason: baseline CDT operations are sufficient; path expressions would add version dependency without materially improving latency." This prevents ambiguity about whether the feature was considered and rejected vs. overlooked.

---

## 8) Sizing, growth, and hot-key guardrails

Define guardrails up front:

- Target record size band for normal traffic (typically 1-128 KiB where practical).
- **Absolute max record safety threshold**, derived from the namespace's configured `max-record-size` — not a fixed constant. State the configured value, the safety threshold you will design to (meaningfully below it, since defrag and I/O cost rise well before the hard stop), and what happens when a record crosses that threshold. If the configured value is unknown, see the `max-record-size` required input in section 0.
- List/map growth triggers for split/overflow/shard.
- Batch-size bounds for key fan-out operations.
- Hot-key detection threshold (write error rate/latency/KEY_BUSY signals). For the mitigation pattern, see [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern.

If you cannot state the trigger values, the model is not deployment-ready.

---

## 9) Validation and failure-mode test plan

Run model validation with representative synthetic data before implementation freeze.

### Required validation cases

- Typical cardinality and skewed cardinality (including outliers).
- Read latency for primary access paths at expected concurrency.
- Write latency and contention for hot paths.
- Growth simulation over retention horizon (record size trajectory).
- Batch path limits (fan-out, payload size, retries).

### Required failure-mode cases

- Partial multi-record update failure (with and without transactions).
- Filtered-out conditional writes under race.
- Hot-key behavior and sharding fallback.
- Delete/cascade correctness under retries and partial completion.
- Version-gated feature unavailable (fallback path exercise).

### Acceptance gate

Declare pass/fail thresholds (latency, error rate, size limits, reconciliation lag).

No pass criteria means no production readiness.

---

## 10) LLM modeling guardrails (generic)

Use these guardrails whenever an LLM is performing data-model design or requirements-to-model translation.

### 10.1) Ambiguity halt rule (mandatory)

If requirements define behavior but leave storage, keying, consistency, or lifecycle implementation ambiguous:

- stop and mark `BLOCKED_MISSING_INPUT`,
- ask targeted clarification questions,
- do not finalize a schema or pattern choice until required inputs are complete.

### 10.2) Clarification protocol (decision-critical only)

Ask for only the information needed to select patterns with confidence:

- cardinality and skew assumptions,
- read/write mix and latency targets,
- retention and lifecycle constraints,
- consistency and correctness requirements,
- growth horizon and operational thresholds.

Question framing and default behavior:

- Phrase questions in project/product language, then translate internally to pattern criteria.
- Avoid "which data model do you prefer?" unless requirement-level constraints still leave multiple valid options after deterministic gates.
- When deterministic guidance yields a default, state the default and why, then ask one exception-check question instead of an open-ended preference prompt.

### 10.3) Requirements vs solution boundary

When editing requirements:

- capture product intent, constraints, and priorities only,
- do not encode physical implementation choices unless explicitly requested.

### 10.4) Deterministic decision record

For each major pattern decision, require:

- inputs used,
- calculations or measured evidence,
- selected option,
- rejected alternatives and why,
- fallback trigger and migration path,
- baseline option,
- proposed option,
- driver type (`requirement-driven` | `evidence-driven` | `speculative`),
- promotion-evidence-present (`yes`/`no`),
- disposition (`adopt_now` | `test_first` | `reject` | `defer`),
- rollback path.

### 10.5) Default-assumption discipline

When required inputs are unavailable:

- use only approved defaults,
- mark assumptions as provisional,
- attach confidence and required follow-up evidence,
- do not present provisional decisions as final.

### 10.6) Conflict-resolution order

When source artifacts conflict:

- Project stakeholder-confirmed contracts override family defaults in referenced guidance documents (e.g., `id-selection-guidance.md` section 6). When a conflict exists, log the override and use the stakeholder-confirmed value.
- apply a fixed precedence order,
- explicitly log unresolved conflicts,
- block final pattern selection until material conflicts are resolved.

### 10.7a) Premature-optimization prevention gate (mandatory)

Before changing baseline model shape (new set, split model, extra SI, materialized projection), record:

- **Baseline option:** current canonical/default shape.
- **Proposed option:** structural change being considered.
- **Driver type:** `requirement-driven` | `evidence-driven` | `speculative`.
- **Required evidence (if not requirement-driven):**
  - p50/p95/p99 cardinality and skew for the target relationship/object,
  - p95/p99 latency or contention evidence showing baseline risk,
  - operational cost tradeoff (write amplification, storage/index overhead, cleanup complexity).
- **Benchmark requirement for read/write tradeoffs:** if the change swaps read-time vs write-time cost (for example, fan-in vs fan-out), attach a minimal comparative test plan and success/fail thresholds.
- **Disposition:** `adopt_now` | `test_first` | `reject` | `defer`.

Rules:

- `speculative` cannot be `adopt_now`.
- missing required evidence => `BLOCKED_MISSING_INPUT`.
- if canonical baseline exists, default disposition is `defer` unless promotion criteria are met.
- if benchmark requirement applies and test plan is missing, force disposition `test_first` (not `adopt_now`).

### 10.7) Parity and exception gate

If a canonical baseline model exists:

- align by default,
- document any deviation with measurable justification, risks, and rollback path.
- include a one-line "why baseline is insufficient" statement.
- if that statement cannot be supported by requirements or measurements, reject the divergence.

### 10.7b) Schema drift resolution gate (mandatory)

Before accepting a divergence or merging artifacts into one contract, resolve schema drift explicitly:

- field naming alignment (for example `actor` vs `actor_handle`),
- value-type alignment (for example string vs integer timestamp),
- unit/format alignment (for example `created_at` vs `created_at_ms`),
- required/optional alignment (including default/omitted behavior),
- TTL/retention representation alignment (record policy vs explicit expiry bin).

Required output:

- a schema-normalization record listing canonical field names/types/units,
- migration/compatibility handling for any renamed or retyped fields,
- explicit decision: `normalized_now` or `blocked_drift`.

Rule:

- if any material schema drift remains unresolved, mark `BLOCKED_MISSING_INPUT` and do not finalize implementation-ready contract status.

### 10.7c) Rejected-optimization logging gate (mandatory)

For each major pattern decision, log rejected optimizations explicitly.

If a decision log mechanism already exists (savepoint, journal entry, ADR note, or equivalent), use that existing artifact instead of creating a new one.

Minimum fields per rejected optimization:

- option rejected,
- reason rejected,
- evidence required to reconsider,
- reconsider trigger/threshold.

Rule:

- if rejected optimizations are not logged in an existing or designated decision artifact, mark `BLOCKED_MISSING_INPUT` and do not finalize implementation-ready contract status.

### 10.8) Validation-before-lock rule

No final recommendation without at least one synthetic validation pass for:

- contention and hot-key behavior,
- growth trajectory and size thresholds,
- delete/cascade and failure-mode correctness.

### 10.9) No hidden requirement inference

LLM may suggest options, but must not silently convert inferred preferences into mandatory constraints.

### 10.10) Optional concrete example

Example (generic): requirements state that one read surface is heavily read while write events are relatively infrequent.  
This is a product priority signal, not a storage prescription. The modeler still must run deterministic pattern gates, document assumptions, and keep architecture choices in the model decision record.

### 10.11) Generic ID-selection guidance and template

Use this section to choose identifier formats deterministically across entities and relationships.
For full normative guidance and project-family defaults, also read:
`id-selection-guidance.md`.

#### ID-selection rules (generic)

- **Primary fork: cleartext composite vs hash.** For each entity ID, first decide whether the components appear in cleartext (human-readable, variable length) or are hashed (compact, fixed size). This is the primary decision — see `id-selection-guidance.md` section 1.1. Only if hash is chosen, proceed to choose which hash (UUID/ULID vs deterministic short hash).
- If hash is chosen, default to UUIDv4 unless explicitly told otherwise. xxHash64 is a valid optimization when repetition pressure justifies space savings.
- Prefer deterministic IDs when idempotent recompute across clients is required.
- Prefer human-readable cleartext composites when repetition pressure is low and operational readability is the higher priority.
- If deterministic hashing is used, lock algorithm, seed, canonical input format, output encoding/length, and collision policy.
- Do not apply UUID/ULID as a blanket default to all entity IDs. Select per entity using repetition pressure, immutable tuple availability, and idempotency requirements.
- Do not finalize ID format without documenting tradeoffs and thresholds.

#### ID decision template (required for major entity IDs)

For each major entity ID (and any edge ID), capture:

- **ID scope:** entity/relationship name.
- **Generation mode:** deterministic-hash / readable-composite / globally-unique-random.
- **Canonical input (if deterministic):** immutable fields and exact string format.
- **Output format:** encoding, length, and allowed character set.
- **Repetition pressure:** where this ID appears (keys, lists, maps, edges) and expected p95/p99 repetition counts.
- **Size impact estimate:** projected bytes contributed by this ID at p95/p99.
- **Collision posture:** detect/reject/remap behavior and operational handling.
- **Cross-client contract:** algorithm/version/seed compatibility requirements.
- **Fallback/migration:** how to evolve format without breaking readers/writers.

#### ID decision gate

If repetition pressure and size impact are not quantified, mark `BLOCKED_MISSING_INPUT` and do not lock the ID format.
If collision handling and canonicalization details are missing for deterministic IDs, mark `BLOCKED_MISSING_INPUT` and do not lock the ID format.

### 10.12) Timestamp and bin naming contract guidance

Use this section to avoid schema drift in timestamp bins and naming conventions.
For full normative guidance, also read:
`timestamp-bin-naming-guidance.md`.

#### Timestamp/bin naming rules (generic)

- Every timestamp field must lock type, unit, and mutability in the contract.
- If using epoch milliseconds, use `_ms` suffix consistently (`created_at_ms`, `updated_at_ms`, `publish_date_ms`).
- Do not mix `_ms` integer bins with string/ISO bins for the same semantic field.
- If a non-`_ms` format is used, document exact encoding, timezone, precision, and rationale.
- Bin names are semantic contracts; do not keep alias variants for the same field across artifacts.

#### Timestamp/bin decision template (required for major fields)

For each major timestamp field, capture:

- **Semantic field:** what time this represents.
- **Bin name:** canonical name.
- **Type/unit:** example `int64 epoch_ms`.
- **Mutability and trigger:** immutable/mutable and when writes occur.
- **Producer and normalization:** server/client source and conversion rule.
- **Migration posture:** dual-read/dual-write/cutover if changing existing format.

#### Timestamp/bin decision gate

If timestamp type/unit/mutability are not explicit for major fields, mark `BLOCKED_MISSING_INPUT` and do not lock the contract.

### 10.13) Do not fabricate required inputs

**Do NOT substitute estimated, inferred, or invented values for missing required inputs.** An LLM's estimate of "p95 ~5000 likes" is not a confirmed input — it is a guess that may drive the wrong pattern selection. If a required input for a decision pack is not stated in the requirements or confirmed by the stakeholder, it is MISSING.

- Mark it `BLOCKED_MISSING_INPUT`.
- Do not record a tentative pattern.
- Do not proceed with sizing math using the fabricated value.
- Do not present the estimate as if it were data.

This rule exists because LLMs will confidently fabricate plausible numbers rather than admit they don't know — and those fabricated numbers will silently drive real design decisions.

**Approved assumptions** are different from fabrications. An approved assumption is a value the stakeholder has explicitly agreed to use as a provisional input (e.g., "assume p95 ~5000 likes for now; revisit after load testing"). The approval, the value, and the follow-up plan must all be documented. An assumption the LLM invents without stakeholder approval is a fabrication, not an approved assumption.

### 10.14) Pre-modeling self-audit (mandatory before writing spec)

Before writing any section of the data model specification, the LLM must produce (internally or as a document section) an audit that lists:

1. Every required input for every decision pack used in this model.
2. For each input: the confirmed value and source, OR `FABRICATED` if the LLM invented it.
3. For each `FABRICATED` input: the question that should have been asked.

If any input is marked `FABRICATED`, the model is not ready. Convert fabricated inputs to `BLOCKED_MISSING_INPUT` and ask the questions.

This step exists as a last-chance catch for the failure mode where the LLM reaches the spec-writing phase without having completed clarification. It forces the LLM to distinguish between "I know this from the requirements" and "I made this up."

### 10.15) Execution cadence discipline (mandatory)

Each entity group in the Step 2.5 plan is a separate interaction unit. The LLM must:

- Complete one group's substeps (a) through (f) before starting the next group.
- Present the implementer review (5.9.2) findings and triage, then the stakeholder review checkpoint (5.9.3) as an explicit pause point — output the checkpoint summary and wait for the stakeholder's response before continuing.
- Not pre-decide pattern choices for groups it has not yet entered. The Step 2.5 plan routes groups to packs and tracks status; it does not lock designs.
- Not batch-execute multiple groups between stakeholder interactions, even if the LLM believes the choices are obvious.

The stakeholder checkpoint is not a formality. It exists because group-specific subtleties surface during design that were not visible at planning time. Assumptions that seemed safe in the plan may be wrong once the sizing gate, contention rubric, or developer walkthrough is applied to concrete data.

**Anti-pattern: monolithic execution plan.** Do not plan or pre-decide record design for all entity groups before entering the per-group loop. A plan that pre-specifies "Group E uses consolidated hierarchy, Group G uses one-record-per-event" has already made the decisions that the per-group gates exist to validate. The correct plan names the groups, routes them to packs, and lists confirmed vs missing inputs — nothing more.

**Why this matters for LLMs specifically.** LLMs are biased toward planning everything upfront because generating a complete plan feels like thoroughness. In an interactive modeling workflow, this bias causes the LLM to treat the stakeholder review checkpoints as pro-forma confirmation of decisions already made, rather than as genuine decision points where the stakeholder might redirect the design. The per-group loop is designed to surface information that changes decisions; pre-deciding the outcomes defeats that mechanism.

---

## 11) Final readiness gate

A new app data model is ready only when:

- All required outputs in section 0 exist and are reviewed.
- Consolidation vs split decisions include explicit "why not the other way."
- Version gates are locked to real environment versions.
- Features-not-used declaration is present for all advanced features in section 7 that the model does not rely on.
- Validation and failure tests are executed and meet thresholds.
- Operational playbook exists for growth, hot keys, and reconciliation.

### 11.1) Final acceptance gate (post-generation, mandatory)

Before accepting a generated data model as implementation-ready, verify all checks below:

- **Contract completeness:** namespace, sets, keys, bins, indexes, TTL/retention, and consistency mode are fully specified and cross-client compatible.
- **Access-path coverage:** every required read/write path maps to concrete operations (single get, batch, bounded query, or justified alternative); no orphan requirements.
- **Invariant mapping:** uniqueness, counters, delete/cascade behavior, visibility/filtering rules, and reconciliation/transaction boundaries are explicitly mapped to operations.
- **Growth and hot-key readiness:** numeric thresholds and fallback paths are documented for size growth, contention, and overflow/sharding transitions.
- **Ambiguity closure:** unresolved material assumptions are zero. If not zero, label output as draft and not implementation-ready.
- **Validation linkage:** each major risk has a corresponding validation or failure-mode test case with pass/fail thresholds.

If any check fails, the model is not accepted.

For detailed patterns and examples, continue with:

- [concepts-and-patterns.md](concepts-and-patterns.md)
- [one-to-many-relationships.md](one-to-many-relationships.md)
- [follow-relationship-scale.md](follow-relationship-scale.md)
- [expressions.md](expressions.md)
- [path-expressions.md](path-expressions.md)
