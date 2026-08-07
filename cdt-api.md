# Aerospike Collection Data Type (CDT) API — Reference

**Summary:** Reference for the List and Map APIs, nested context, and ordering/comparison rules from the [Aerospike documentation](https://aerospike.com/docs/develop/data-types/collections/). Use alongside `concepts-and-patterns.md` when designing or implementing data models.

**Status:** Reference.

**Source docs:**
- [Collections overview](https://aerospike.com/docs/develop/data-types/collections/)
- [List](https://aerospike.com/docs/develop/data-types/collections/list/)
  - [List performance](https://aerospike.com/docs/develop/data-types/collections/list/performance/)
  - [List examples](https://aerospike.com/docs/develop/data-types/collections/list/examples/)
  - [List indexing and querying](https://aerospike.com/docs/develop/data-types/collections/list/index-and-query/)
- [Map](https://aerospike.com/docs/develop/data-types/collections/map/)
  - [Map performance](https://aerospike.com/docs/develop/data-types/collections/map/performance/)
  - [Map examples](https://aerospike.com/docs/develop/data-types/collections/map/examples/)
  - [Map indexing and querying](https://aerospike.com/docs/develop/data-types/collections/map/index-and-query/)
- [Nested context](https://aerospike.com/docs/develop/data-types/collections/context/)
- [Order and compare](https://aerospike.com/docs/develop/data-types/collections/ordering/)

---

## Collections overview

- **CDTs** are List and Map: schema-free containers that can hold scalars or nest other lists/maps. Elements can be mixed types.
- Bins hold one value per bin; that value can be a scalar, a CDT, or another supported type (e.g. HyperLogLog, GeoJSON).
- CDTs are a superset of JSON (e.g. integer map keys, bytes).
- **Multi-op transactions:** multiple list, map, or scalar ops can be combined in a single record operation; policies control rollback vs partial success on error.
- HyperLogLog can live inside a collection; blob/HLL ops inside CDTs use read-/write-expressions.

---

## List API

### List types and terminology

- **Unordered list:** Keeps insertion order as long as elements are appended. Rank-based access uses just-in-time value order.
- **Ordered list:** Keeps value (rank) order; re-sorts on insert. Mixed-type elements ordered by type then value.
- **index** — 0-based position (e.g. index 2 = third element). Negative = from end (-1 = last).
- **rank** — Value order (rank 0 = smallest value). Negative = from end.
- Same API for both; convert with `set_order()`.
- **Ordered lists and `ADD_UNIQUE`:** When using `ADD_UNIQUE` for set-like deduplication, always use an ordered list. On an ordered list, `ADD_UNIQUE` duplicate detection uses binary search (O(log n)). On an unordered list, uniqueness enforcement falls back to a linear scan (O(n)). For relationship or reference-ID lists that may hold thousands of entries, the performance difference is significant.
- **Offset index persistence (`persistIndex`).** Ordered lists can persist the offset index in the list particle. Without persist, the offset index is rebuilt per-operation (O(N) walk through packed elements). With persist, index-based access is O(1). Set via list policy `persistIndex` flag or `set_order` with the persist option. In an ordered list, index and rank are equivalent, so rank-based operations also benefit from the persisted index. Persist-index is only supported for top-level lists; nested list persist-index is silently ignored.

### List operations (representative)

| Category   | Operations |
|-----------|------------|
| Order     | `set_order()`, `sort()`, `clear()` |
| Write     | `append()`, `append_items()`, `insert()`, `set()`, `increment()` |
| Size      | `size()` |
| By index  | `get_by_index()`, `get_by_index_range()` |
| By rank   | `get_by_rank()`, `get_by_rank_range()` |
| By value  | `get_all_by_value()`, `get_all_by_value_list()`, `get_by_value_interval()`, `get_by_value_rel_rank_range()` |
| Remove    | `remove_by_index()`, `remove_by_index_range()`, `remove_by_rank_range()`, `remove_all_by_value()`, `remove_all_by_value_list()`, `remove_by_value_interval()`, `remove_by_value_rel_rank_range()` |

- List bin is created when a list value is written to a bin or when using list `append`, `insert`, `set`, or `increment`.
- Insertion at either end is fast; append/delete at end is generally efficient.
- **Limitations:** Bound by max record size. List commands are not supported in Lua UDFs.

### List performance characteristics

Worst-case complexity for modeling-relevant operations. N = element count, M = range/interval result count, R = min(r, N-r-1) for rank r.

| Operation | Unordered | Ordered | Ordered + Persisted Index |
|-----------|-----------|---------|--------------------------|
| `get_by_index` | O(N) | O(N) | O(1) |
| `get_by_index_range` | O(N + M) | O(N + M) | O(M) |
| `get_by_value_interval` / `get_all_by_value` | O(N + M) | O(log N + N) | O(log N + M) |
| `get_by_rank` † | O(1) | O(R log N + N) | O(N) |
| `get_by_rank_range` | O(R log N + N) | O(N + M) | O(M) |
| `append` / `insert(0)` / `set(0)` | O(1) | O(log N + N) | O(log N) |
| ADD_UNIQUE append/insert | O(N) | O(log N + N) | O(log N) |
| `remove_by_index` | O(N) | O(N) | O(1) |
| `remove_by_value_interval` / `remove_all_by_value` | O(N + M) | O(log N + N) | O(log N + M) |
| `remove_by_rank_range` | O(R log N + N) | O(N + M) | O(M) |

**† `get_by_rank` anomaly.** These are the figures published in the [list performance table](https://aerospike.com/docs/develop/data-types/collections/list/performance), reproduced as documented. The profile is counterintuitive — it makes the unordered list the cheapest and the persisted-index list worse than O(1) — and it inverts the pattern every other row follows. Treat it as unconfirmed: benchmark before designing around single-rank access on a list, and prefer `get_by_rank_range` (whose profile is conventional) where it can serve the same need. Worth raising with the docs team.

**Legend:** Unordered = no internal indexes. Ordered = value order maintained, offset index rebuilt per operation. Ordered + Persisted Index = offset index stored in the list particle. In an ordered list, index and rank are equivalent, so persisted-index benefits apply to rank operations as well. Every modify op has an additional copy-on-write cost for rollback. Storage ops add +D (load from storage) and +W (write to storage).

**Return-type performance implications.** The complexities above assume the default return type. Non-default return types add CPU cost. For example, requesting RANK on index-range operations against an unordered list adds O(L*N) where L = min(i, N-i-1).

### Secondary index on list elements

A secondary index (SI) on a list bin creates one index entry per list element. Elements are type-checked against the index data type (numeric, string, or GeoJSON); non-matching types are skipped.

- List indexing supported at any depth (DB 6.1.0+; prior versions: top-level only).
- Known limitation: range queries on list-indexed bins can return duplicate records when a list contains multiple values that fall within the query range.

---

## Map API

### Map types and terminology

Aerospike Maps have three subtypes that differ in how elements are ordered and what internal indexes they maintain. All three share the same API and support key, index, value, and rank based operations. They can be converted between each other with `set_type()`.

**From DB 7.0:** all Maps are stored in key order on the server regardless of the order hint the client used when creating them. The subtype therefore describes **which internal indexes the map maintains**, not whether the bytes on storage happen to be sorted. "Unordered" means no index and no *guaranteed* order the application may rely on — not that the server stores elements in arbitrary order. An application that depends on the informal ordering of an unordered map can break if that ordering changes.

- **Unordered:** Elements have no guaranteed order. No internal indexes; all lookups scan elements. Lowest storage overhead.
- **K-ordered:** Elements are stored in key order. Has a key offset index that maps each key position to a byte offset within the packed map.
- **KV-ordered:** Elements are stored in key order (same as K-ordered). Has both a key offset index and a value order index that maps rank to element. The value order index enables efficient rank-based and value-based operations.

- **key** — Element identifier. **value** — Element value. **index** — Position in key order (0 = smallest key). **rank** — Position in value order (0 = smallest value); ties broken by index (the element earlier in key order gets the lower rank).
- **From DB 7.1.0:** Map keys restricted to **integer, string, blob**.

**Index persistence (`PERSIST_INDEX`).** By default, internal indexes are rebuilt temporarily for each operation and discarded afterwards. Setting `PERSIST_INDEX` in the map policy stores the indexes on disk inside the map particle, so subsequent operations load them directly instead of rebuilding them. There are two persistence levels:

- **Persisted offset index:** Any map subtype with `PERSIST_INDEX` (without `V_ORDERED`) persists the key offset index. Eliminates the per-operation O(N) index rebuild cost. Enables O(log N) key lookups and O(1) index-based access.
- **Persisted full index:** Any map subtype with `PERSIST_INDEX` and the `V_ORDERED` flag persists both the key offset index and the value order index. Enables O(1) rank lookups and O(log N) value-based searches in addition to offset-index benefits.

**`PERSIST_INDEX` on an unordered map.** Setting `PERSIST_INDEX` on an unordered map sorts its keys into key order and persists the offset index, giving it the same binary-search key-lookup behavior as a K-ordered map with a persisted index (column 3 of the performance table). So "unordered + persist" is not a distinct performance tier — it converges on K-ordered + persist. Choosing unordered to save storage overhead and then persisting the index gives up that saving.

Persist-index is only supported for top-level maps. Nested map persist-index is silently ignored. KV-ordered without a persisted index has no performance advantage over K-ordered without a persisted index for rank and value operations — both fall back to a heap/scan. The advantage of KV-ordered without persist is that converting to KV-ordered with persist is O(1) if elements are already in value order.

**Equality comparison caveat.** Only ordered maps (K-ordered or KV-ordered) can be reliably compared for equality. Unordered maps have no canonical byte ordering, so two unordered maps with the same logical content can have different wire representations. Comparisons involving unordered maps — whether through an expression `eq` operator or through `*_by_value` / `*_by_value_list` operations — may return false even when the maps contain the same elements. This matters for list-of-maps patterns using `ADD_UNIQUE`.

#### Worked example: index and rank with tied values

Map: `{a:1, b:2, c:30, y:30, z:26}` (K-ordered, so stored in key order: a, b, c, y, z).

| Element | a:1 | b:2 | c:30 | y:30 | z:26 |
|---------|-----|-----|------|------|------|
| key     | a   | b   | c    | y    | z    |
| value   | 1   | 2   | 30   | 30   | 26   |
| index   | 0 or -5 | 1 or -4 | 2 or -3 | 3 or -2 | 4 or -1 |
| rank    | 0 or -5 | 1 or -4 | 3 or -2 | 4 or -1 | 2 or -3 |

**How rank is computed:** Sort elements by value using the type-then-value ordering (see Order and compare below). Assign rank 0 to the smallest, rank N-1 to the largest. When two elements have the same value, the one with the lower index (earlier in key order) gets the lower rank.

Walk-through: values sorted ascending are 1, 2, 26, 30, 30. `a:1` → rank 0. `b:2` → rank 1. `z:26` → rank 2. `c:30` and `y:30` tie on value; `c` is at index 2, `y` is at index 3, so `c:30` → rank 3, `y:30` → rank 4.

Negative indexing counts from the end: `map_get_by_index(-1)` returns `z:26` (last in key order); `map_get_by_rank(-1)` returns `y:30` (largest value, tie broken by highest index).

#### Map index vs rank: when to use which

- **Index-based access** (by key order): Use when the access dimension is the key itself — key lookups, key-range scans, "first N" or "last N" by key, pagination over keys. Always efficient on K-ordered maps.
- **Rank-based access** (by value order): Use when the access dimension is the value — top-N by value, value-range filtering (e.g. "all entries with value ≥ threshold"), application-level expiry by value.
- **Composite-key technique:** When the modeler needs rank semantics but wants to leverage key-order efficiency, encode the sort dimension in the map key (e.g. `score-playerId` so key order = score order). Map index then equals rank within the map, and index-based operations serve as rank operations. This is used in leaderboard and sorted-structure patterns. See [concepts-and-patterns.md](concepts-and-patterns.md) Source 6 (Leaderboards) for the full pattern.

#### Serving multiple sort dimensions

Map rank serves **one** value-based sort dimension natively (scalar value, or the leading element of a list value via list comparison). When the application requires additional sort dimensions on the same consolidated structure — e.g., both time order and score order — build one auxiliary sorted structure per additional dimension:

- **Auxiliary ordered list** for time order: append IDs on create; read forward for oldest first, reverse for newest first.
- **Auxiliary map** for score/rank order: `{ id: score }` with `map_put` on mutation; `get_by_rank_range` for top-N.

Each auxiliary is maintained on write (add/remove) alongside the primary structure. Only dimensions not already served by the primary structure's native order require auxiliaries. Because multiple ops in a single `operate()` execute atomically under the record lock, the primary structure and all auxiliaries stay consistent — no multi-record transaction is needed. See [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § "Ordering contract: sort-dimension pattern" for the full escalation ladder and a worked example.

#### Worked example: rank with list values (user segment expiry)

When map values are lists, rank is determined by list comparison: element-by-element from index 0, then by length (see Value comparison below). This means the first element of each list drives the rank order — a pattern used in data modeling to enable value-range operations on a chosen sort dimension.

**User segment map:** Each map entry represents a segment for a user. Key = segment ID (integer). Value = `[ttl, attributes]` where `ttl` is an integer (e.g. hours since a fixed epoch) and `attributes` is a map of per-segment metadata.

```
{101: [4800, {src: "web"}], 205: [3600, {}], 310: [5100, {src: "app"}]}
```

Rank is determined by comparing the list values. List comparison starts at index 0, so `ttl` (the first element) drives the order:

| Element | 205: [3600, {}] | 101: [4800, {src: "web"}] | 310: [5100, {src: "app"}] |
|---------|-----------------|---------------------------|---------------------------|
| index   | 1               | 0                         | 2                         |
| rank    | 0 (lowest ttl)  | 1                         | 2 (highest ttl)           |

**Why this matters for modeling:**

- **Get non-expired segments:** `map_get_by_value_interval(bin, [now, NIL], [INF, INF])` returns all entries with `ttl ≥ now`. The interval bounds are lists so Aerospike compares list-to-list; `NIL` is the lowest value in any position (see Type order below) and `INF` is the highest.
- **Trim expired segments:** `map_remove_by_value_interval(bin, [0, NIL], [now, NIL])` removes all entries with `ttl < now` in a single server-side operation.
- **Structure the list so the sort dimension is first.** The `[ttl, attributes]` layout puts the value that drives rank/range operations at list index 0. If `attributes` were first, rank would sort by the attributes map (using map comparison rules) rather than by TTL.

See [concepts-and-patterns.md](concepts-and-patterns.md) Source 2 (User Profile Store) for the full applied pattern including background scan trim.

### Map operations (representative)

| Category   | Operations |
|-----------|------------|
| Type      | `set_type()` |
| Write     | `put()`, `put_items()`, `increment()`, `decrement()`, `clear()` |
| Size      | `size()` |
| By key    | `get_by_key()`, `get_by_key_interval()`, `get_all_by_key_list()` |
| By index  | `get_by_index()`, `get_by_index_range()`, `get_by_key_rel_index_range()` |
| By value  | `get_by_value_interval()`, `get_by_rank_range()`, `get_all_by_value()`, `get_all_by_value_list()`, `get_by_value_rel_rank_range()` |
| Remove    | `remove_by_key()`, `remove_by_key_interval()`, `remove_by_index()`, `remove_by_index_range()`, `remove_by_value_interval()`, `remove_by_rank_range()`, `remove_all_by_value()`, etc. |

### Map op flags and return types

- **INVERTED** — Inverts the selection (e.g. “remove all but the 10 largest by index”).
- **Return types:** None, Index, RevIndex, Rank, RevRank, Count, Key, Value, KeyValue.

### Map guidelines

- Multiple ops can be combined in one record command. Map bin created on `put`/`put_items`/`increment` or when a map value is written to the bin.
- All three map subtypes support key, index, value, and rank operations. The subtypes differ in which internal indexes they maintain, affecting performance. See Map performance characteristics below.
- For maps accessed frequently, use `PERSIST_INDEX` to store internal indexes on disk rather than rebuilding them on each access.
- Nested list/map ops: from DB 4.6.0. Relative rank ops: from 4.3.0.
- **Limitations:** Bound by max record size. Map commands are not supported in Lua UDFs.

### Map performance characteristics

Worst-case complexity for modeling-relevant operations. N = element count, M = range/interval result count, R = min(r, N-r-1) for rank r, L = min(i, N-i-1) for index i.

| Operation | Unordered | Ordered (no persist) | Persisted Offset Index | Persisted Full Index |
|-----------|-----------|---------------------|----------------------|---------------------|
| `get_by_key` | O(N) | O(N) | O(log N) | O(log N) |
| `get_by_index` | O(L log N + N) | O(N) | O(1) | O(1) |
| `get_by_index_range` | O(L log N + N) | O(N + M) | O(M) | O(M) |
| `get_by_rank` | O(R log N + N) | O(R log N + N) | O(R log N) | O(1) |
| `get_by_key_interval` | O(N) | O(N) | O(log N + M) | O(log N + M) |
| `get_by_value_interval` / `get_all_by_value` | O(N + M) | O(N + M) | O(N + M) | O(log N + M) |
| `get_by_rank_range` | O(R log N + N) | O(R log N + N) | O(R log N) | O(M) |
| `put` | O(N) | O(N) | O(log N) | O(log N) |
| `increment` / `decrement` | O(N) | O(N) | O(log N) | O(log N) |
| `remove_by_key` | O(N) | O(N) | O(log N) | O(log N) |
| `remove_by_rank` | O(R log N + N) | O(R log N + N) | O(R log N) | O(1) |
| `remove_by_rank_range` | O(R log N + N) | O(R log N + N) | O(R log N) | O(M) |

**Legend:** Unordered = no internal indexes, linear scan. Ordered (no persist) = K-ordered or KV-ordered with offset index rebuilt per operation. Persisted Offset Index = key offset index stored in map particle (K-ordered + persist, or any subtype + persist without `V_ORDERED`). Persisted Full Index = both key offset and value order indexes stored (any subtype + persist with `V_ORDERED`). Every modify op has an additional copy-on-write cost for rollback. Storage ops add +D (load) and +W (write).

**Return-type performance implications.** The complexities above assume the default return type. Key-based operations add cost when returning RANK results (sped up by persisted full index). Value-based operations add cost when returning INDEX results (sped up by persisted full index). The `ORDERED_MAP` return type adds cost only when the source map is unordered.

**Worked example: top-N leaderboard.** A map `{name: score}` with KV-ordered + persisted full index. `get_by_rank_range` for the top 10 scores is O(M) = O(10) = constant time (plus +D for storage load). Without persist, the same operation is O(R log N + N) — linear in the map size.

**Worked example: paginating a map by key.** Paging through a consolidated map with `get_by_index_range(offset, count)` is O(M) — proportional to the page size, not the map — once the offset index is persisted. Without persist it is O(N + M) on an ordered map and O(L log N + N) on an unordered one, so every page read walks the whole map and pagination cost grows with the consolidated record rather than the page. This is the main reason to set `PERSIST_INDEX` on any map that backs a paginated read path, and it mirrors the list case (`list_get_by_index_range`, O(M) with `persistIndex`). Pagination bounds the **response**; the persisted index is what bounds the **work done to produce it**.

### Secondary index on map keys/values

A secondary index (SI) on a map bin creates one index entry per map element. SI can index on map keys (`MAPKEYS` source type) or map values (`MAPVALUES` source type). Elements are type-checked against the index data type (numeric, string, or GeoJSON); non-matching types are skipped.

- Map indexing supported at any depth (DB 6.1.0+; prior versions: top-level only).
- Known limitation: range queries on map-indexed bins can return duplicate records when a map contains multiple values that fall within the query range.

---

## Nested context

- **Context** identifies the path to the element an operation targets. Top-level = no context.
- Context is a list of (type, value) selectors applied from the top level or from the previous step.

### Depth contract (modeling guardrail)

When modeling nested Lists/Maps (CDTs), define a depth contract before finalizing schema:

- platform max nesting/path depth (for your DB/version/features),
- application max depth (with safety margin),
- behavior at cap (`reject`, `split`, or `overflow`).

If platform limit or app cap is unknown, mark `BLOCKED_MISSING_INPUT` and do not finalize the model.

### Context types

| Type                 | Description |
|----------------------|-------------|
| `BY_LIST_INDEX(idx)` | List element at index |
| `BY_LIST_RANK(rank)` | List element at rank |
| `BY_LIST_VALUE(val)` | List element by value. Selects the **first** matching element in list (index) order when duplicates exist. Requires an exact value — WILDCARD is not allowed. |
| `BY_MAP_INDEX(idx)`  | Map element at index |
| `BY_MAP_RANK(rank)`  | Map element at rank |
| `BY_MAP_KEY(key)`    | Map element by key |
| `BY_MAP_VALUE(val)`  | Map element by value. Selects the **first** matching element in index order when duplicates exist. Requires an exact value — WILDCARD is not allowed. |
| `MAP_KEY_CREATE(key)`   | Create map key if missing, then select (4.9+) |
| `LIST_INDEX_CREATE(idx)` | Create list slot if missing, then select (4.9+) |

- **CDT context** feature: DB 4.6.0+. Create-if-missing context types: 4.9.0+.
- **Each selector must identify exactly one element**, and must target an element whose type matches the operation. Applying a list operation to a scalar or a map (or vice versa) returns error 26 (`OP_NOT_APPLICABLE`). To select and operate on multiple elements at once, use path expression contexts instead.

### Path expression context types (DB 8.1.1+)

The context types above each select a single element at each level. Path expressions extend this model to select and filter multiple elements at once, using `selectByPath` and `modifyByPath` instead of traditional single-element CDT operations. These context types are used with path expression operations only, not with standard CDT operations.

**Matching and filtering (8.1.1+):**

| Type | Description |
|------|-------------|
| `ALL_CHILDREN` | Matches all children of the current Map or List without filtering. |
| `ALL_CHILDREN_WITH_FILTER(exp)` | Matches children where the filter expression evaluates to true. The filter can assign an aspect of the current element (value, key, or index) to a loop variable. |

**Key selection and combined filtering (8.1.2+):**

| Type | Description |
|------|-------------|
| `MAP_KEYS_IN(keys...)` | Select map entries whose keys match any of the provided values. Equivalent to SQL `WHERE key IN (k1, k2, ...)`. Uses the map's internal index for efficient lookup. |
| `AND_FILTER(exp)` | Apply an additional filter expression at the same level as the preceding context. Entries must satisfy both the preceding context and this filter. |

**`AND_FILTER` constraints:**

- One expression per context level — `AND_FILTER` cannot be chained after another `AND_FILTER`.
- Cannot be used after `ALL_CHILDREN` or `ALL_CHILDREN_WITH_FILTER`.
- Not supported inside expression-wrapped operations (`CdtExp.selectByPath`). Use direct CDT operations (`CdtOperation.selectByPath` / `CdtOperation.modifyByPath`) instead.

### Example: drilling into a list

List: `[0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]`

- Target `[3, 4]` with context `[BY_LIST_INDEX(2), BY_LIST_INDEX(1)]` (third top-level element, then second element of that list).
- Then e.g. `list_append('l', 100, context=[BY_LIST_INDEX(2), BY_LIST_INDEX(1)])` appends 100 to `[3, 4]`.

### Example: operating on map value (list)

- Map: segment_id → `[ttl, attrs]`. To increment the TTL (first list element) at a given segment_id, use **list** op with context **by map key**: e.g. `list_increment("u", 0, 5, ctx=[cdt_ctx_map_key(segment_id)])`.

### Example: create path then operate

- `map_increment('m', 'jokes', 317, context=[MAP_KEY_CREATE('stats'), MAP_KEY_CREATE('accolades')])` creates `stats` and `accolades` if missing, then increments `jokes` under `accolades`.

---

## Order and compare

### Type order (ascending)

1. NIL  
2. BOOLEAN  
3. INTEGER  
4. STRING  
5. LIST  
6. MAP  
7. BYTES  
8. DOUBLE  
9. GEOJSON  
10. INF  

- Different types compare by type first; same type by value. Maps use this ordering for value-based (rank) operations.
- **NIL:** singleton, lowest type; used as the lower bound in list and map interval and range operations. It **can** be stored as a list element or a map value and reads back successfully, but it **cannot be used as a map key**. Assigning NIL to a top-level bin **deletes that bin** — and if it is the record's last bin, the record is removed. Two consequences for modeling: never let a nullable field become a top-level bin whose "empty" state is written as NIL, and when a map value is legitimately absent, storing NIL inside the CDT is safe while a NIL map key is not.
- **INF** (4.3.1+): singleton, highest type; used in intervals, not for storage. Storing INF in a list or map has undefined behavior.

### Value comparison

- **List:** Compare element by element from index 0; then by length. E.g. `[1,2] < [1,3]`, `[1,2] < [1,2,1]`.
- **Map:** By element count, then keys in stored order, then values when keys match. (Pre-4.3.1: known issues for different-length maps/lists.) Map comparison requires a canonical byte ordering, which only ordered maps (K-ordered or KV-ordered) provide. Unordered maps cannot be reliably compared for equality — see the equality comparison caveat in Map types and terminology above.

**Map comparison worked example:** Given two maps A = `{x:10, y:20}` and B = `{x:10, y:30}`:

1. Compare element count: both have 2 elements → tie, proceed.
2. Compare keys in stored (K-ordered) order: both have keys `x`, `y` in the same order → tie, proceed.
3. Compare values where keys match: at key `x`, both have value 10 → tie; at key `y`, A has 20, B has 30 → A < B.

Result: `{x:10, y:20} < {x:10, y:30}`. If A had 3 elements and B had 2, A > B regardless of key/value content (element count wins first). This ordering applies when maps appear as values inside a list (list-of-maps pattern) and rank or value-interval operations are used on that list.

### Wildcard and intervals

- **Wildcard (`*`):** Valid only as a value parameter in list and map `*_by_value` / `*_by_value_list` operations. When used as an element in a list pattern, it matches **any remaining elements from that position onward** — not just the one position. Given bin `[[1,1],[1,2],[1,3],[2,1],[2,2]]`, `list_get_all_by_value([1, *])` returns `[[1,1],[1,2],[1,3]]`. Because the match runs to the end of the tuple, a pattern like `["v2", *]` also matches longer tuples such as `["v2", 4, {"a": 1}]`, and matches tuples of size two or more. Cannot be used in a CDT context selector (`BY_LIST_VALUE` / `BY_MAP_VALUE`), which must identify exactly one element. Not a storage type — storing WILDCARD has undefined behavior. 4.3.1+.
- **Intervals:** Default **inclusive-exclusive** `[start, end)`.
- **INF** for inclusive end: e.g. `list_get_by_value_interval(start=[1, NIL], end=[2, INF])` gives elements ≥ `[1, NIL]` and ≤ `[2, INF]` at the second level. 4.3.1+.

---

## Relation to data modeling (this project)

- **User profile store:** Bin is a **map** (segment_id → value). Value = **list** `[ttl, attrs]`. Range ops use **list-to-list** comparison: e.g. `map_get_by_value_range` with `[now, null]`..`[infinity, null]` for “non-expired” segments. **Context** `BY_MAP_KEY(segment_id)` used with **list_increment** to update TTL in place.
- **IoT sensors:** Bin is a **list** of `[minute, temp_x10]`. **get_by_value_interval** for time ranges; ordering and interval rules apply.
- When comparing list or map values in range ops, wrap scalars in a list if the stored value is a list (e.g. segment value `[ttl, {}]` compared with `[ttl, null]` or wildcard).
