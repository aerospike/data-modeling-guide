# Aerospike Data Modeling: One-to-Many Relationships

**Summary:** Patterns for modeling one-to-many relationships in Aerospike: when to keep a list on the parent, when to consolidate children into one record per parent, and when to put the link on the child side and use a secondary index. Choice is driven by **cardinality and who drives the read**. Record size should follow the **Goldilocks Principle** (1–128 KiB ideal; avoid too small for index-to-data ratio, too large for I/O and defrag). Informed by project data models and applied research.

**Status:** Research. Complements [concepts-and-patterns.md](concepts-and-patterns.md) (Goldilocks Principle, primary index cost).

---

## 1. Parent-held list of child keys

The parent record stores a list of child IDs (e.g. user has `post_ids`; post has `repost_ids`).

**When it fits:** The "many" side has **modest cardinality** and you typically load the parent when you need the children (e.g. "list my posts", "list reposts of this post").

**Pros:** One get on the parent gives you the keys; then batch get children. No secondary index. Simple.

**Cons:** The parent record grows with the number of children. If that list gets very large, the parent record is dominated by it and you may hit record-size or write-amplification concerns.

**Rule of thumb:** Use when you're confident the list stays small (e.g. reposts per post; posts per user within your scale).

---

## 2. Consolidation: one record per parent that contains all children

Instead of N child records, you have **one record per parent** keyed by parent id (e.g. `post_comments` keyed by `post:{id}`). All "children" live inside that record as a nested map/list (e.g. comment tree plus `comment_paths`, `comment_order`, etc.).

**When it fits:** **High cardinality** and **small children** (e.g. hundreds of comments per post, each comment small).

**Pros:** One primary index entry per parent (not per child); one get returns the whole set; no N keys to maintain on the parent; avoids blowing up the parent record.

**Cons:** That record can get large (must stay under limits); updates to one child are in-place operates with path/context; you need a way to target a single child (e.g. `comment_paths` for comments). When children are consolidated, inverse lookups ("find all parents containing child X") require additional infrastructure — either an SI on a distinct-authors/child-IDs list, a per-child reverse tracking structure, or async scan. See the [checklist's](new-app-modeling-checklist.md) Decision Pack 3 inverse-lookup closure step for the required pattern selection.

**Rule of thumb:** Prefer when the "many" would otherwise be a long list on the parent or many small records, and you're happy to read/update in one record.

---

## 3. Child-held parent reference + secondary index (inverse lookup)

Children (or a record that aggregates them) hold the parent id or a "who's here" list (e.g. `commenters` on `post_comments`). You create a secondary index on that bin and **query by the "many" side** (e.g. "all posts where this user commented").

**When it fits:** You need **inverse access** ("find all X that reference this Y") and you don't want the referenced entity to hold a growing list (e.g. user must not hold every post they commented on).

**Pros:** The referenced entity (e.g. user) stays small; you get a bounded set of records to touch (e.g. for user-delete cascade).

**Cons:** Inverse lookups are queries (secondary index) plus batch get/operate, not a single get on the parent.

**Rule of thumb:** Use when cardinality of the "many" is high and you need to find "all parents (or containers) that reference this one thing" (e.g. all post_comments records where this user commented).

---

## 4. How to choose: cardinality and who drives the read

| Situation | Pattern |
|-----------|---------|
| **Low cardinality, parent-driven reads** | List on parent (e.g. `repost_ids`). |
| **High cardinality, parent-driven reads** | Consolidate into one record per parent (e.g. comments in `post_comments`). |
| **High cardinality, child-driven or inverse reads** | Don't put the list on the "one" side; put the link on the "many" (or its container) and use a secondary index to query (e.g. `commenters` for user-delete). |
| **Very high cardinality (N exceeds single-record capacity), parent-driven reads** | Companion record with ordered list of reference IDs, sharded into sub-records when N exceeds the ~128 KiB target. See section 5 (companion record overflow). |

**CDT sub-element operations are parent-driven access.** When children are consolidated into a parent record and accessed via CDT operations (e.g., `map_put` on one child entry, `list_append` to a nested list), this is still parent-driven access for the purpose of pattern selection — not child-driven access. The operation executes within the parent record's I/O path. Child-driven access means the dominant path is direct single-child read/write by an independent child key, without loading the parent.

**Record size (Goldilocks Principle):** Aim for records in the **1–128 KiB** band. Too small and the 64-byte primary index per record dominates (bad when PI is in memory); too large and read/write plus defrag hurt performance. See [concepts-and-patterns.md](concepts-and-patterns.md) § Foundational concepts.

**Child payload size matters as much as count.** The pattern 1 / pattern 2 boundary is not purely about cardinality — it depends on **child count x average child size**. Tiny children (reference IDs, handles at ~15 bytes) can be consolidated by the thousands and stay within the Goldilocks band. Large children (full documents at several KiB each) push past the band at much lower counts. Always compute the aggregate before choosing.

### Worked sizing example — consolidation vs. split

Suppose a post can have up to ~600 comments (p99), each averaging ~300 bytes (short text plus a few metadata fields). Access is parent-driven: "load the comment thread for this post."

**Step 1 — Aggregate size:** 600 comments × 300 bytes = **~175 KiB**. This is modestly above the 128 KiB Goldilocks target but well within the configured `max-record-size`, even at its 1 MiB default.

**Step 2 — Compare alternatives:**

| Approach | Records | PI cost | Read cost | Verdict |
|----------|---------|---------|-----------|---------|
| One record per comment | 600 records | 600 × 64 bytes = **37.5 KiB** of index for ~175 KiB of data | 600-record batch get + client-side ordering | Bad PI ratio; ordering complexity |
| Consolidate into one record per post | 1 record | 1 × 64 bytes | One get returns the whole thread | Good PI ratio; single-record access; record size (~175 KiB) is acceptable |

**Consolidation wins** — the aggregate fits one record at a manageable size, and the access pattern is parent-driven.

**Step 3 — Sensitivity to child size.** Now suppose each child is a rich document averaging **5 KiB** instead of 300 bytes:

- 600 × 5 KiB = **~3 MiB** — beyond too large for comfortable single-record I/O: this exceeds the configured `max-record-size` on any default namespace, so the write is rejected outright. At this child size, prefer separate child records (pattern 1: list of child keys on the parent, batch get children) or sharded consolidation (section 5).

The crossover is roughly: `child_count_p99 × avg_child_bytes ≈ 128 KiB`. Below that, consolidation is the default. Above that, evaluate split records, companion overflow, or sharded consolidation depending on access pattern and how far above the band the aggregate falls. See [concepts-and-patterns.md](concepts-and-patterns.md) § Worked example for the full comments-on-a-post walkthrough and [follow-relationship-scale.md](follow-relationship-scale.md) § Sizing for a reference-list scenario at 300k–10M scale.

### Mixed access: when one pattern is not enough

The decision table above assumes the access pattern is purely parent-driven or purely child-driven. Many relationships have **both**: a dominant parent-driven read path **and** an inverse path needed for lifecycle operations or secondary views. Choosing consolidation (pattern 2) for the parent-driven path does not eliminate the need for inverse infrastructure (pattern 3). Both may be required simultaneously.

**Example 1 — Comments on a post (consolidation + inverse SI).**

- **Parent-driven path:** "Load comment thread for post X." Served by the consolidated `post_comments` record keyed by `post:{id}`. One get returns the full thread.
- **Inverse path:** "User deletes account — find and redact all their comments across all posts." This requires locating every `post_comments` record that contains comments by the deleted user. Without inverse infrastructure, the only option is a full-namespace scan.
- **Solution:** Maintain a `commenters` list bin on each `post_comments` record (distinct author handles, ordered, ADD_UNIQUE). Create a secondary index on `commenters`. On user-delete, query the SI to find all `post_comments` records containing the user, then redact within each via CDT operations. This combines pattern 2 (consolidation for the parent-driven read) with pattern 3 (SI on the consolidated record for the inverse lifecycle path).

**Example 2 — Likes on a post (consolidation + per-user reverse index).**

- **Parent-driven path:** "Show who liked post X." Served by a `post_likes` record (consolidated ordered list of user handles keyed by `post:{id}`). One get returns the liker list.
- **Inverse path:** "Show all posts this user has liked" (profile page). This requires finding all `post_likes` records containing the user — an SI query would work, but the access pattern is by known user key, not an ad-hoc search.
- **Solution:** Maintain a per-user `user_liked_posts` companion record (ordered list of post IDs keyed by user handle). On like, write to both `post_likes` and `user_liked_posts`. "Posts I liked" is a single get by user key — no SI needed. This combines pattern 2 (consolidation for the parent-driven path) with a reverse companion record for the user-driven path.

**After choosing the primary pattern for the dominant read path, check every remaining access path and lifecycle operation** — especially user-delete cascade, moderation actions, and profile-page views. If any requires the inverse direction, add the minimal inverse infrastructure. Each inverse structure adds write amplification (one extra write per child-create to maintain the SI list or reverse index), which is the trade-off for avoiding full scans on the inverse path. See [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 5.6 inverse-lookup closure for the three supported inverse patterns (SI-backed parent lookup, per-child reverse index, approved async scan) and their cost/risk trade-offs.

### Write contention as a pattern-selection factor

Cardinality, child payload size, and access direction determine the **structural** pattern choice. Write contention determines whether the chosen structure can **sustain the expected write rate** without degradation.

Aerospike applies a record-level lock on every write. When multiple concurrent writes target the same record, later arrivals may receive `KEY_BUSY` (error code 14) and must retry. On a consolidated record, every child-create, child-update, and child-delete is a write to the same parent record. The larger the record, the longer each write holds the lock (read-modify-write of the full record). High concurrent writes on a large consolidated record compound both effects: lock duration increases with record size, and lock contention increases with write rate.

**When contention matters for pattern selection:**

- A consolidated `post_comments` record receiving occasional new comments (a few per minute even on a popular post) is firmly in **Band A** — contention is negligible and consolidation is the clear default.
- A consolidated `post_likes` record on a viral post receiving hundreds of concurrent likes per second can push into **Band B or C** — the record becomes a write hot spot. At this point, consolidation alone is insufficient. Options: shard the likes record via shard-on-demand (hash-shards distribute concurrent writes across sub-records), or move to per-like records if the like payload is trivial and PI cost is acceptable.
- A `user_followers` companion record for a celebrity gaining thousands of followers during a viral moment hits the same problem at larger scale. The [follow-relationship-scale doc](follow-relationship-scale.md) § Sizing covers this: pre-shard for users known to be high-fanout, and use hash-shards so concurrent follows distribute across sub-records.

**Rule of thumb:** If the expected concurrent write rate on a single consolidated record exceeds ~50 writes/second sustained (or bursts significantly higher), evaluate the contention risk before committing to consolidation. The [checklist's contention rubric](new-app-modeling-checklist.md) § 5.2 provides the formal Band A/B/C classification using `lambda_window`, record size, and observed `KEY_BUSY` rate. The [shard-on-demand pattern](concepts-and-patterns.md) § Shard-on-demand is the standard mitigation when contention pushes a consolidated record into Band B or C.

---

## 5. Companion record overflow for large N

When the "many" side of a 1:N relationship is too large for a list bin on the parent entity but the access pattern is still parent-driven (not child-driven), move the list into a **companion record** keyed by the parent. When N grows beyond what a single companion record can hold, apply the **shard-on-demand pattern** (see [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern) to distribute the list across S sub-records that preserve the same list-based structure.

### 5.1 Baseline: single companion record

The companion record holds an **ordered** list of reference IDs (child keys or handles) in a single list bin.

- **Key:** derived from the parent (e.g., set `user_followers`, key = parent handle).
- **List policy:** `LIST_ORDERED`, `ADD_UNIQUE`, `NO_FAIL`. Ordered storage is required — on an ordered list, `ADD_UNIQUE` uses binary search for duplicate detection (O(log n)); on an unordered list, uniqueness enforcement requires a linear scan (O(n)). For relationship lists that may grow to thousands of entries, ordered lists are required for acceptable write latency.
- **Offset index persistence (`persistIndex`):** For companion lists expected to hold thousands of entries, set `persistIndex` on the list policy. Without persist, the offset index is rebuilt per-operation (O(N) walk through packed elements). With persist, index-based reads (pagination via `list_get_by_index_range`) are O(M) for M returned elements instead of O(N + M), and `ADD_UNIQUE` inserts are O(log N) instead of O(log N + N). persist-index is only supported for top-level lists (the companion list bin is top-level, so this applies). See [cdt-api.md](cdt-api.md) § List performance characteristics.
- **Target max record size:** ~128 KiB. Deduce the safe element count from ID length and record overhead: `safe_count ≈ 128 KiB / avg_id_bytes`. When projected p99 element count exceeds this, plan for overflow from the start.

At this stage, all reads and writes are normal CDT operations on a single record. No extra routing logic is needed.

### 5.2 Overflow via shard-on-demand

When the companion record crosses the ~128 KiB target, apply the shard-on-demand pattern from [concepts-and-patterns.md](concepts-and-patterns.md): write a `subkeys` bin to the companion record. Writes use a filter expression that checks for the absence of `subkeys`; on FILTERED_OUT the application reads `subkeys` to determine routing and retries against the correct sub-record.

Each sub-record uses the same list structure as the baseline companion: ordered list, `ADD_UNIQUE`, `NO_FAIL`, with `persistIndex` when element counts are expected to reach thousands.

### 5.3 Bucket strategy selection

Choose the `subkeys` tuple based on access shape:

| Access shape | `subkeys` value | Routing | Sub-record key |
|---|---|---|---|
| Even write distribution / membership queries | `["hash-shards", S]` | `child_id % S` | `{companion_key}:{i}` for i in 0..S−1 |
| Recent-window reads (e.g., "latest followers") | `["hours", S]` | Time buckets of S hours, anchored to midnight UTC | `{companion_key}:{bucket_timestamp}` |
| Coarse time partitioning (e.g., daily batches) | `["days", S]` | Time buckets of S days, anchored to midnight UTC | `{companion_key}:{bucket_timestamp}` |

`["hash-shards", S]` is the default for reference-list overflow. Time-based types (`"seconds"`, `"minutes"`, `"hours"`, `"days"`) are appropriate when reads are strongly biased toward recent entries and older shards can be skipped entirely.

Read: get the companion record. If `subkeys` exists, read `[type, S]` and batch read the sub-records — all S for hash-shards, or the relevant time range for time types. Concatenate the ordered lists.

### 5.4 Overflow transition (application-triggered)

1. **Detect threshold crossing.** The application detects that the companion record has exceeded the target size — either via filtered-write failure (if a size-based filter is used) or via periodic size monitoring.
2. **Write `subkeys`.** Write `subkeys = ["hash-shards", S]` (or the chosen time type) to the companion record. From this point, new writes are routed to sub-records via the filter-expression mechanism.
3. **Create sub-records.** Create sub-records with the same structure (ordered list bin, same list policy).
4. **Redistribute entries.** Move existing entries from the companion's list into sub-records by `child_id % S` (for hash-shards) or by timestamp (for time types). Use bounded batches to avoid large single operations.
5. **Clean up the original list.** Remove the list bin from the companion record. The companion now serves as a routing stub (`subkeys` present, no list data).

Writes are safe during transition because the filter expression blocks writes to the companion as soon as `subkeys` is added. Reads should check for `subkeys` and fall back to sub-record batch reads.

### 5.5 Re-shard trigger

When a sub-record itself outgrows the target threshold: for hash-shards, increase S, update `subkeys` on the companion, and redistribute affected sub-records. For time types, narrow the time granularity (e.g., change from `["hours", 6]` to `["hours", 1]`) instead of increasing S.

Document the re-shard trigger threshold and the expected migration procedure as part of the data model contract.

### 5.6 Migration and rollback

**Forward migration (list-on-parent to companion to sharded companion):** Each step is additive. The parent entity can drop its list bin once the companion record is live. The companion can add `subkeys` once sub-records are live.

**Rollback (sharded companion to single companion):** Merge all sub-records back into a single companion list, remove `subkeys`. This is safe when N has shrunk below the single-record threshold (e.g., after a bulk unfollow or data cleanup).

**Rollback (companion to list-on-parent):** Move the list back to the parent entity's bin, delete the companion record. This is safe when N has shrunk to modest cardinality.

Document the rollback trigger and procedure as part of the data model contract.

---

## References

- [Aerospike Data Modeling — Research Notes](concepts-and-patterns.md) — Foundation, indexes, and applied patterns.
- [New-app modeling checklist](new-app-modeling-checklist.md) — Decision packs for relationship pattern selection; overflow/shard triggers in the required decision records.


