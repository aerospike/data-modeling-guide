# Modeling failure modes

The eight ways Aerospike data models most often go wrong. These are not LLM-specific — they are relational and document-database habits applied to an architecture that rewards neither, and human modelers make all of them.

Use this file two ways:

- **Priming**, before designing. Read the **Rule** and **Detect** lines; they are what you check *while* drafting.
- **Review rubric**, against a drafted model. Each **Detect** line is phrased as a test you can run over a proposed schema and get a yes/no answer.

Each entry is tagged:

- **Portable** — holds without the rest of this guide. Safe to lift into a skill or an agent instruction file.
- **Guide-only** — meaningful only inside this guide's workflow. Do not lift; it will not be actionable out of context.

Values that a cluster operator can change (`max-record-size`) or that are gated on a server version are stated **by name** here and given concrete numbers only in the linked reference files. Do not copy such numbers into downstream skills — they drift, and a stale copy is worse than a pointer.

---

## 1. Defaulting to one record per sub-entity

**Rule.** Decide record granularity from cardinality and from who drives the read — never from the entity list.

**Detect.** Count the sets in the draft. If each maps 1:1 to a domain noun (comment, follow, event, like), the model was derived from an ER diagram rather than from access patterns.

**Why.** Every record costs 64 bytes of primary index metadata, usually in RAM. A million tiny records spend more memory on index than on data.

**Instead.** Often consolidate — one record per parent holding many children in a CDT, or a dedicated consolidated record keyed by parent. But this is a tradeoff, not a rule: for **inverse access** ("all parents where this child appears") or very large fan-out, a child-held reference plus a secondary index may be correct. **Skew matters** — most entities have small relationships while a few have enormous ones, so pick the pattern that fits the bulk case and tolerates outliers, typically consolidated storage with paginated list reads. → [one-to-many-relationships.md](one-to-many-relationships.md), [follow-relationship-scale.md](follow-relationship-scale.md)

**Tier.** Portable.

---

## 2. Using secondary indexes as the primary query mechanism

**Rule.** The most frequent reads must resolve to a key lookup or a bounded batch read.

**Detect.** List every access pattern and mark how each resolves. If more than one or two resolve via secondary-index query rather than key lookup or batch read, the key design is wrong — fix the keys, not the indexes.

**Why.** SI queries scatter to every node and cannot match a direct get-by-key or batch-get. Each index also carries per-entry memory cost, and a collection-typed bin produces one entry *per element*, not per record.

**Instead.** Design keys and denormalized lists so hot reads are key lookups. Reserve SIs for inverse lookups and for queries where the key set is genuinely unknown in advance. For unique-ID resolution, use a lookup-table record rather than an index. → [concepts-and-patterns.md](concepts-and-patterns.md)

**Tier.** Portable.

---

## 3. Ignoring CDT capabilities

**Rule.** Mutations that touch one element of a collection must happen server-side, in place.

**Detect.** Trace each write. Any operation that reads a bin, changes part of it in application code, and writes the whole bin back is a read-modify-write that a CDT operation should have replaced.

**Why.** Lists and maps support append, remove-by-value, get-by-value-range, get-by-index-range and more, at arbitrary nesting depth via context. Pulling a collection to the client to edit it spends network and latency on work the server does under the record lock — and loses atomicity.

**Instead.** Use the List and Map operation APIs with nested context; combine several in one `operate()` call so they apply atomically. → [cdt-api.md](cdt-api.md)

**Tier.** Portable.

---

## 4. Treating bins like columns

**Rule.** A bin is a container, not a field. Model repeating or sparse data as one CDT bin, not many scalar bins.

**Detect.** Look for bin counts that scale with data rather than with schema — a bin per tag, per day, per counter. Also flag any bin name that had to be truncated past readability.

**Why.** A bin can hold a list, a map, a nested structure, or a HyperLogLog. One bin holding a map of a thousand entries is generally better than a thousand single-value bins, which carry higher metadata overhead and, past roughly a hundred bins, a bin-merge cost that grows quadratically on write.

**Instead.** Consolidate into CDT bins. Bin names are limited to **15 characters**: start descriptive and abbreviate only on hitting the limit, keeping type-carrying suffixes (`_ms`, `_cnt`) intact. → [cdt-api.md](cdt-api.md), [workload-archetypes.md](workload-archetypes.md) (Archetype B2)

**Tier.** Portable.

---

## 5. Normalizing instead of denormalizing

**Rule.** Duplicate data deliberately when two access patterns need it in two shapes.

**Detect.** Find any read that must fetch a second record purely to assemble a response — a follower count read from the follower-list record, a display name read from the user record. Each is a normalization the model should have collapsed.

**Why.** There are no server-side joins. The alternative to duplication is a second round trip on the read path, and reads usually outnumber the writes that must maintain the duplicate.

**Instead.** Store the same value in both places when it serves different patterns — a follower count on the user record *and* the follower list in a consolidated record. Write both in one `operate()` where they share a record; where they do not, record the reconciliation strategy explicitly. → [concepts-and-patterns.md](concepts-and-patterns.md)

**Tier.** Portable.

---

## 6. Unbounded collection growth

**Rule.** Every list or map bin needs a defined growth ceiling and a decided behavior at that ceiling.

**Detect.** For each CDT bin, ask what caps its element count. If the answer is user behavior rather than a design decision — followers, comments, events, notifications — it is unbounded. Ask for p99 cardinality three years out; if nobody can answer, that is a `BLOCKED_MISSING_INPUT`, not a detail to settle later.

**Why.** Record size is bounded by the namespace's **`max-record-size`**, which is a *configurable* limit with a default well below its permitted maximum — a model sized against the maximum will fail on a default-configured namespace with error 13 (`AS_ERR_RECORD_TOO_BIG`). Long before that hard stop, large records raise defragmentation cost and I/O latency, because each write rewrites the whole record.

**Instead.** Consolidation with paginated reads, dedicated consolidated records, threshold-triggered overflow, or sharding. Pagination bounds the **response**; you still need a storage pattern that bounds the **record**, and a persisted collection index so the work of producing each page is bounded too. → [follow-relationship-scale.md](follow-relationship-scale.md), [concepts-and-patterns.md](concepts-and-patterns.md) § Record size limits — the only place in this guide that states the actual default and ceiling

**Tier.** Portable.

---

## 7. Treating the checklist as a document template

**Rule.** Design one entity group at a time, and pass its gates before starting the next.

**Detect.** If a complete schema exists for every entity group and no clarifying question was asked and no stakeholder checkpoint was held, the workflow was skipped regardless of how complete the document looks.

**Why.** The checklist defines a process with mandatory stop points — group-specific clarification, blocker gates, stakeholder review. Reading it, extracting its headings, and using them as the outline of a single-pass document produces a model that looks compliant but was never validated at any intermediate stage.

**Instead.** Work the per-group loop: clarify, design, walk through, review, update status. → [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 5.9, § 5.9.1

**Tier.** Guide-only — this describes this guide's workflow and is not actionable without it.

---

## 8. Ignoring PI cost for small independent entities

**Rule.** An entity with no relationships and a small payload still needs an explicit sizing decision.

**Detect.** Find sets whose records are a few hundred bytes and that participate in no relationship. Multiply the expected population by 64 bytes and compare against the payload total. If index cost is a double-digit percentage of data, it is unresolved — and a single consolidated record for all of them is the opposite error.

**Why.** 64 bytes of index for a 300-byte payload is roughly 21% overhead, paid in RAM, at every replica. Over-consolidating to fix it creates a hot key that serializes writes.

**Instead.** Choose by population and access pattern: modest populations stay one record per entity; large populations use hash-bucket or domain-grouped consolidation to reach the low end of the 1–128 KiB band — single-digit KiB is the target here, not the upper end; or move index cost off memory entirely with primary index on flash (All Flash). → [concepts-and-patterns.md](concepts-and-patterns.md) (Data modeling tips, "Small independent entities below Goldilocks")

**Tier.** Portable.
