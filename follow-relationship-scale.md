# Follow Relationships at Scale: Modeling N:M Follows in Aerospike

**Summary:** A social service lets users follow each other. Each user can post short messages; all followers see those messages in a feed. The baseline data model stores `followers` and `following` as lists on the user record. That design does not scale when users have hundreds of thousands of followers (or follow many accounts). This note describes the problem, why a one-record-per-follow set is a bad fit (primary index ratio), and a **consolidated** scalable alternative that stores each direction in a dedicated record keyed by user.

**Status:** Research. Informs design if the product expects very popular users (e.g. hundreds of thousands of followers).

**See also:** [concepts-and-patterns.md](concepts-and-patterns.md) (record sizing, 64-byte PI cost, few KiB = 1–128 KiB Goldilocks band); [one-to-many-relationships.md](one-to-many-relationships.md); [new-app-modeling-checklist.md](new-app-modeling-checklist.md) (Decision Pack 2: N:M pattern selection, PI cost comparison).

---

## Problem with lists on user

Today the user record has:

- **`followers`** — list of handles (unique human-readable identifiers, e.g. `@alice`) of users who follow this user.
- **`following`** — list of handles this user follows.

Access patterns: feed (read my `following`, batch get those users' `post_ids`); follow/unfollow (append/remove from both users' lists); user delete (get my followers, remove me from each of their `following`, clear my lists).

**At scale:** A very popular user (e.g. a celebrity) might have hundreds of thousands of followers (300k × ~15 bytes ≈ 4.5 MB in one list). The user record would dominate or exceed practical record size. So follow data must move off the user record.

---

## Why not one record per follow?

A naive move is a **follow set**: one record per (follower, followed) pair, key `follow:A:B`, bins follower + followed, with secondary indexes for "followers of B" and "who does A follow."

**Primary index ratio:** Aerospike incurs **~64 bytes of primary index metadata per record** (see [concepts-and-patterns.md](concepts-and-patterns.md)). A follow record is tiny: key ~40–60 bytes, two short string bins ~25–35 bytes → **~50–150 bytes per record**. So we'd pay **64 bytes of index in RAM for a ~100-byte record** — a terrible ratio. At millions of follow relationships, index memory would dominate. The research guidance is to **consolidate data into the few-KiB sweet spot (1–128 KiB; see [concepts-and-patterns.md](concepts-and-patterns.md) § Terminology)** where practical, and avoid many tiny records that inflate primary index cost. The consolidated follow design may exceed that band when the alternative is worse PI ratio; list pagination keeps the application path bounded.

**Secondary index cost on top:** The two SIs needed for this design ("followers of B" and "who does A follow") add **~14 bytes per SI entry** (see [concepts-and-patterns.md](concepts-and-patterns.md) § Secondary index). At 100M follow relationships, that is ~2.6 GiB of SI memory (× replication factor) — on top of the already-bad PI cost. SI queries also scatter to all nodes, making them slower than a single get-by-key. The consolidated design eliminates both SIs entirely.

---

## Consolidated alternative (one record per parent)

The principle is the same as any consolidated one-to-many pattern: one record per **parent** holding a list of **children**, instead of one record per child. (See [one-to-many-relationships.md](one-to-many-relationships.md) section 2.) Follows are N:M (many followers, many followees), but the standard Aerospike approach decomposes N:M into **bidirectional key lists** (see [concepts-and-patterns.md](concepts-and-patterns.md) § N:M). Here that means two independent 1:N consolidations — one per direction — because each direction has independent cardinality and skew (a user may have 300k followers but follow only 500 accounts).

For follows, apply this in **two directions** (we need both "followers of B" and "who does A follow" by key):

1. **Followers of a user** — One record per user who has ≥1 follower. Key = that user's handle (e.g. set `user_followers`, key = B). Bin = **handles** (list of handles). "List followers of B" = one **get by key**. One 64-byte PI per user who has followers; the record can be large (e.g. 300k handles ≈ 4.5 MB). Ratio: 64 bytes index per 4.5 MB record.

2. **Who a user follows** — One record per user who follows ≥1. Key = that user's handle (e.g. set `user_following`, key = A). Bin = **handles** (list of handles). "Who does A follow?" / feed = one **get by key**. One 64-byte PI per user who follows someone; the record can be large (e.g. 10k handles ≈ 150 KB). Ratio: 64 bytes index per record.

So we **denormalize** the relationship into two lists, but store those lists in **dedicated records keyed by one side**, not on the user record and not as one record per follow. Record count and PI count are **O(users with follow activity)**, not O(follow relationships). No secondary index is required for "list followers of B" or "feed" — both are get by key.

### Sets and keys (consolidated)

| Set              | Record key          | Bin              | Purpose                                                                    |
| ---------------- | ------------------- | ---------------- | -------------------------------------------------------------------------- |
| `user_followers` | followed_handle (B) | `handles` (list) | One record per user who has ≥1 follower. Get by key = list followers of B. |
| `user_following` | follower_handle (A) | `handles` (list) | One record per user who follows ≥1. Get by key = who does A follow; feed.  |

List policies: ORDERED, ADD_UNIQUE, NO_FAIL, with `persistIndex` for lists expected to grow into the thousands. Without persist, the offset index is rebuilt per-operation (O(N) walk); with persist, pagination reads via `list_get_by_index_range` are O(M) for M returned elements instead of O(N + M), and `ADD_UNIQUE` inserts are O(log N) instead of O(log N + N). persist-index is only supported for top-level lists (the companion list bin is top-level). See [cdt-api.md](cdt-api.md) § List performance characteristics.

User record: remove `followers` and `following` bins; keep `follower_cnt` (and optionally `following_cnt`). Create each record on first add; delete when list becomes empty if desired.

### Access pattern drives the design

**What the UI needs:** (1) **Counts** — "How many followers does this user have?" and "How many does this user follow?" (2) **Paginated lists** — "Show this user's followers" and "Show who this user follows," in pages (e.g. 20 or 50 per page). We do **not** need to return a full list in one query.

So the data model can rely on:

- **Aggregated counts on the user record** (`follower_cnt`, optionally `following_cnt`) for the count display. One read of the user gives both numbers.
- **Consolidated list records** (user_followers, user_following) that can grow large, with **list API pagination** (e.g. `list_get_by_index_range` or equivalent) to fetch a slice of followers or following at a time. The UI never needs the full list in one response.
- **Infrequent writes** — Follow/unfollow are list append or list_remove with ORDERED, ADD_UNIQUE, NO_FAIL. These records get big but are not written on every request; the write pattern is manageable.
- **User delete** — The lists are **ordered**; we pop groups of elements off the **front** using **list_remove_by_index_range**(0, batch_size) repeatedly. For each batch: (1) **list_get_by_index_range**(0, batch_size) to read the handles in that slice; (2) **list_remove_by_index_range**(0, batch_size) to remove that batch from the list (front is always index 0 after each pop); (3) for each handle in the batch, operate list_remove_by_value(me) on that user's opposite record (followers → their `user_following`; following → their `user_followers`) and decrement counts. No need to load an entire multi‑MB list; we process in bounded batches. Same for both my `user_followers` and my `user_following` lists. (Note: the second argument to `list_get_by_index_range` and `list_remove_by_index_range` is the **count**, not the end index.)

### Access patterns

| Operation                  | How                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Follow (Alice → Bob)**   | Operate list_append on Bob's `user_followers` record (create with empty list if missing). Operate list_append on Alice's `user_following` record (create if missing). Operate increment follower_cnt on Bob; optionally increment following_cnt on Alice.                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Unfollow**               | Operate list_remove_by_value on both records; operate decrement counts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Feed**                   | Get `user_following` by key (my_handle). Read handles. Batch get user records (post_ids) for those handles. Build feed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **List followers of Bob**  | Get `user_followers` by key (Bob). Use list API (e.g. list_get_by_index_range) to paginate over handles; return one page at a time to the UI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **List who Alice follows** | Get `user_following` by key (Alice). Use list API to paginate over handles; return one page at a time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **User delete**            | **Followers:** While my `user_followers` list is non-empty: list_get_by_index_range(0, batch_size) to get the next batch of handles; list_remove_by_index_range(0, batch_size) to pop that batch off the front (ordered list, so front stays at index 0); for each handle in the batch, operate list_remove_by_value(me) on that user's `user_following` and decrement their following_cnt. **Following:** Same on my `user_following` list — pop front in batches with list_remove_by_index_range(0, batch_size), then remove me from each of those users' `user_followers` and decrement follower_cnt. Delete my two records. Proceed with rest of user delete. |

User delete is still O(N+M) operates (N = followers, M = following), but we are not allocating 64 bytes of index per follow relationship; we are updating existing consolidated records.

---

## Trade-offs

| One record per follow                                             | Consolidated (two lists keyed by user)                                                                                                       |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| 64 bytes PI per ~100-byte record; index memory dominates at scale | 64 bytes PI per user-with-activity; record holds list (exceeds few-KiB band at scale — see Sizing below for overflow thresholds); good ratio |
| Two SIs for "followers of B" and "who A follows"                  | No SI; both paths are get by key                                                                                                             |
| Put/delete one record per follow/unfollow                         | Two operates (append/remove on two lists) per follow/unfollow                                                                                |
| User delete = two SI queries + delete records                     | User delete = two gets + N+M operates to remove me from others' lists                                                                        |

---

## Multi-record consistency

Follow and unfollow each touch 2+ records: the followed user's `user_followers` record, the follower's `user_following` record, and counter updates on both user records (up to 4 record mutations per operation). User delete is O(N+M) mutations across many records.

If one write succeeds and another fails, the two sides of the relationship are temporarily inconsistent (e.g. Alice appears in Bob's followers but Bob does not appear in Alice's following, or the count is off by one). How to handle this depends on the namespace's consistency mode:

- **Strong-consistency (CP) namespace:** Use multi-record transactions (Aerospike Database 8+) to update all sides atomically. Simplest correctness guarantee.
- **AP-mode namespace:** Update both sides independently. If relationship consistency matters, verify after writes (read-back check, background reconciliation). Many social applications tolerate brief inconsistency between the two sides — a follower count that is off by one for a few seconds is acceptable.

User delete is inherently long-running and cannot be made atomic across thousands of mutations. Run it as an idempotent batch process: each batch pop + cross-record cleanup is a bounded unit of work. If the process crashes mid-way, restart from whatever remains in the list (the front is always index 0 after each pop). Mark the user as "deleting" before starting so that new follows are rejected.

See [concepts-and-patterns.md](concepts-and-patterns.md) § Multi-record consistency for the general guidance on N:M relationship mutations.

---

## Sizing and overflow thresholds

**Handle size assumption:** ~15 bytes per handle (MessagePack-encoded string). Aerospike list overhead is small relative to element payload at these counts.

**Safe count per record:** The Goldilocks target is ~128 KiB per record. `safe_count ≈ 128 KiB / 15 bytes ≈ 8,700 handles`. A consolidated record with more than ~8,700 handles exceeds the target and should plan for shard-on-demand overflow (see [one-to-many-relationships.md](one-to-many-relationships.md) § 5).

**300k followers (popular user):**

- Record size: 300k × 15 bytes ≈ **4.5 MB** — not merely above the ~128 KiB target but **above the configured `max-record-size` on any default namespace** (default 1 MiB), so the unsharded write is rejected outright with error 13, not just slow. Sharding is required here, not merely advisable.
- Shard count (hash-shards): 300k / 8,700 ≈ **35 sub-records**, each ~128 KiB. Routing: `handle_hash % 35`. Batch read all 35 sub-records and concatenate for paginated list display.
- PI cost: 1 companion record + 35 sub-records = **36 PI entries × 64 bytes = 2.3 KiB** of index for 4.5 MB of data. Ratio remains excellent.

**10M followers (extreme outlier):**

- Record size without sharding: 10M × 15 bytes ≈ **150 MB** — far exceeds any possible `max-record-size`, including the 8 MiB architectural ceiling. Sharding is mandatory.
- Shard count (hash-shards): 10M / 8,700 ≈ **1,150 sub-records**. A batch read of 1,150 records is operationally heavy. Alternatives:
  - **Larger shards:** Increase target shard size to **512 KiB** (reasonable for infrequently-read full-list use cases). `512 KiB / 15 bytes ≈ 34,000 handles per shard`. 10M / 34,000 ≈ **295 sub-records**. Batch read of 295 records is feasible; paginated access only touches a subset. Do not set the shard target at 1 MiB on a default namespace: that is exactly the default `max-record-size`, leaving no headroom for list and record overhead, so shards would fail as they fill. A 1 MiB target requires raising `max-record-size` first.
  - **Time-bucketed shards:** Use `subkeys = ["days", 1]` (one shard per day). New followers land in today's shard; "latest followers" reads only recent shards. Old shards are rarely touched. Shard count grows over time but each shard stays bounded by daily follow rate.
- PI cost at 295 sub-records: 296 × 64 bytes ≈ **19 KiB** of index for ~150 MB of data. Ratio is excellent.
- **Write contention:** At 10M followers, the companion record is a write target for every new follow. If follow rate is high (e.g. hundreds of concurrent follows during a viral moment), the record may hit `KEY_BUSY`. The shard-on-demand pattern handles this: with hash-shards, concurrent writes distribute across sub-records, reducing per-record contention. Pre-shard (`subkeys` set at creation) for users known to be high-fanout. See [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern.

---

## When to adopt

- **Keep current design** (lists on user) if follower and following counts are modest (e.g. most users have hundreds, not hundreds of thousands).
- **Adopt consolidated design** (user_followers + user_following, lists keyed by user) if the product expects very popular users (e.g. hundreds of thousands of followers) or very large follow lists, and you want a single design that scales without a terrible primary-index ratio.
- **If a consolidated record exceeds the ~128 KiB target** (e.g., a celebrity with millions of followers), apply the shard-on-demand pattern to distribute the list across sub-records. See [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern for the mechanism and [one-to-many-relationships.md](one-to-many-relationships.md) § 5 for the full companion overflow pattern (baseline sizing, bucket strategy, transition, rollback).
