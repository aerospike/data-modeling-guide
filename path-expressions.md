# Aerospike Path Expressions — Research Summary

**Summary:** Path expressions enable granular querying and indexing of nested List and Map structures (context stack, loop variables, selectByPath/modifyByPath). Use with [cdt-api.md](cdt-api.md) and [concepts-and-patterns.md](concepts-and-patterns.md) when designing document-style or nested-CDT models.

**Status:** Production. Requires **Aerospike Database 8.1.2** or later (8.1.1 introduced preview support; 8.1.2 adds `mapKeysIn`, `andFilter`, and is the documented production prerequisite). Check [client compatibility](https://aerospike.com/docs/develop/client-matrix).

**Scope and assumptions for new app modeling:** This is a version-gated advanced feature reference. Start with [new-app-modeling-checklist.md](new-app-modeling-checklist.md), confirm DB/client compatibility, and keep a denormalized fallback pattern for production environments where path expressions are unavailable or not accepted.

**Source docs:**

- [Path expressions overview](https://aerospike.com/docs/develop/expressions/path/)
- [Quickstart](https://aerospike.com/docs/develop/expressions/path/quickstart/)
- [Advanced usage](https://aerospike.com/docs/develop/expressions/path/advanced/)
- [Performance](https://aerospike.com/docs/develop/expressions/path/performance/)
- [FAQ](https://aerospike.com/docs/develop/expressions/path/faq/)
- [Troubleshooting](https://aerospike.com/docs/develop/expressions/path/troubleshooting/)
- [Working with nested CDTs (nesting tutorial)](https://aerospike.com/docs/develop/expressions/nesting/)
- [Context types](https://aerospike.com/docs/develop/data-types/collections/context/)

---

## Overview

Path expressions add **contextual expressions** over nested [List](https://aerospike.com/docs/develop/data-types/collections/list/) and [Map](https://aerospike.com/docs/develop/data-types/collections/map/) CDTs. Aerospike provides three complementary ways to work with nested collection data: **CDT operations** (`ListOperation`, `MapOperation`) act in-place on the record at a known position; **CDT expressions** (`ListExp`, `MapExp`) evaluate to a typed value for filtering and computed reads; **path expressions** extend both models by selecting or modifying **multiple elements at once** using expression-based contexts. Path expressions address:

- **Performance:** Server-side filtering and selection so only matching elements (or subtrees) are returned — less network transfer than fetch-whole-record-then-filter.
- **Flexibility:** Native support for JSON-like document models (e.g. map of products → map/list of variants) without denormalizing or writing client-side loops.

Combined with [expression indexes](https://aerospike.com/docs/database/learn/tutorials/quick-start-expression-index/), path expressions allow **indexing and querying** data inside deeply nested structures.

### Key capabilities

- **Multi-element selection:** Select, retrieve, or manipulate multiple elements from a nested CDT in one operation.
- **Iteration + loop variables:** Iterate over map key-value pairs or list elements; use **loop variables** (MAP_KEY, VALUE, LIST_INDEX) in expressions to filter or read from the current element.
- **Expression indexes:** Select multiple elements from a document (nested CDT) to build an index; query and traverse deep structures without denormalization.

---

## Concepts

### Context stack

Path operations use a **context stack** that defines how to traverse the bin's CDT:

1. Start at the bin (e.g. `inventory`).
2. Each step is a **context** (e.g. "all children", "all children matching filter", "map key `variants`").
3. Filters are **expressions** evaluated at each level; they can use **loop variables** to read the current iteration's key, value, or list index.

Example shape (from Quickstart):

```
inventory (bin)
 └── product (map entry, keyed by productId)
     ├── category, featured, name, description
     └── variants  (map of SKU→attrs or list of variant objects)
         └── variant-level filter (e.g. quantity > 0)
```

### Loop variables

When iterating over CDT elements, the server exposes the current element via loop variables so expressions can reference it:

| Loop variable  | Meaning (map)                                | Meaning (list)           |
| -------------- | -------------------------------------------- | ------------------------ |
| **MAP_KEY**    | Key of current entry                         | N/A                      |
| **VALUE**      | Value of current entry (often a map or list) | Element at current index |
| **LIST_INDEX** | N/A                                          | Index in the list        |

Example: "product-level" filter `featured == true` uses `MapExp.getByKey(..., "featured", Exp.mapLoopVar(LoopVarPart.VALUE))` — the loop variable is the product map, and the expression reads its `featured` field.

### Context types

#### Matching and filtering (8.1.1+)

- **allChildren()** — Descend into all children (map entries or list elements).
- **allChildrenWithFilter(filter)** — Same, but keep only elements for which the expression evaluates true.
- **mapKey(key)** — Descend into the map entry for the given key (e.g. `"variants"`).

Same idea as CDT nested context (BY_MAP_KEY, etc.), but used here with **expressions** and **filters** instead of fixed paths.

#### Key selection and combined filtering (8.1.2+)

- **mapKeysIn(keys...)** — Select map entries whose keys match any of the provided values. Equivalent to SQL `WHERE key IN (k1, k2, ...)`. Uses the map's internal index for efficient lookup — O(M log N) for M requested keys in an N-entry ordered map, versus O(N × M) for the `allChildrenWithFilter` + `ListExp.getByValue` workaround available in 8.1.1.
- **andFilter(exp)** — Apply an additional filter expression at the same level as the preceding context. Entries must satisfy both the preceding context and this filter to be included.

**Constraints on mapKeysIn and andFilter:**

- Only **one expression per level**. `Exp.and(exp1, exp2)` counts as one expression.
- `andFilter` **cannot** be chained after another `andFilter` — combine conditions into a single `Exp.and(...)` instead.
- `andFilter` **cannot** follow an `allChildren` or `allChildrenWithFilter` context (those already carry an expression).
- `andFilter` **cannot** be the first context — it filters what the preceding context selects.
- Multiple `mapKeysIn` + `andFilter` **pairs can be chained at successive nesting levels** in a single `selectByPath` call (e.g., select top-level keys, filter, then select inner keys and filter again at the deeper level).

**Example (Java):**

```java
CdtOperation.selectByPath("doc",
    SelectFlags.MATCHING_TREE | SelectFlags.NO_FAIL,
    CTX.mapKeysIn(10001L, 10003L),          // select rooms by ID
    CTX.andFilter(Exp.and(timeFilter, deletedFilter)),
    CTX.allChildren(),
    CTX.allChildrenWithFilter(rateFilter));
```

**Performance comparison** (IN-list filtering on a map with N entries, M requested keys):

| Approach                          | API                                                                             | Complexity | Notes                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- | ---------- | ----------------------------------------------------- |
| Filter-as-you-go (8.1.1+)         | `CdtOperation.selectByPath` with `allChildrenWithFilter` + `ListExp.getByValue` | O(N × M)   | Visits every entry; linear membership check per entry |
| Pre-filter then traverse (8.1.1+) | `CdtExp.selectByPath` wrapping `MapExp.getByKeyList` inside `ExpOperation.read` | O(M log N) | Index-based lookup, but requires expression wrapper   |
| Native key selection (8.1.2+)     | `CdtOperation.selectByPath` with `CTX.mapKeysIn` + `CTX.andFilter`              | O(M log N) | Cleanest API; no expression wrapper overhead          |

---

## Operations

### selectByPath (read)

- **Purpose:** Select a subset of the nested CDT and return it.
- **Parameters:** Bin name, **selectFlags**, and a list of context steps (with optional filters).
- **Result:** Depends on selectFlags (see below).

### modifyByPath (write)

- **Purpose:** Modify or remove selected elements in place.
- **Parameters:** Bin name, modify flags, an **expression**, and the same kind of context stack. The expression determines what happens to each matched element:
  - **Value transformation** — the expression computes a replacement value (often using loop variables to read the current value and `MapExp.put` / similar to write back). Example: increment `quantity` for all in-stock variants of featured products.
  - **Element removal** — pass `ResultRemove` as the expression. The server deletes every matched element from its parent container. Combined with `allChildrenWithFilter`, this provides **selective server-side deletion**: e.g., remove all map entries where `status == "expired"`, or remove list elements that satisfy a predicate — in a single operation with no read-modify-write round trip.
- **Use case:** Bulk updates or bulk deletions on filtered subsets of a nested structure without read-modify-write in the client.

---

## Path expressions in the expression API

Path expressions can be used in two distinct API contexts:

### CdtOperation.selectByPath / modifyByPath (direct CDT operations)

The primary API. Executes a path expression directly as a record operation via `client.operate()`. Supports the full context vocabulary including `mapKeysIn` and `andFilter` (8.1.2+).

### CdtExp.selectByPath / modifyByPath (expression-wrapped operations)

Path expressions embedded **inside** the expression API. Used when:

- **Filter expressions:** A `CdtExp.selectByPath` extracts nested values, and the result feeds into a boolean check (e.g. `ListExp.getByValue(EXISTS, ...)`) that filters records during a query or scan. The path expression runs server-side as part of expression evaluation, not as a standalone operation.
- **Expression index creation:** A `CdtExp.selectByPath` extracts values from nested structures (e.g. all license plates from a list-of-maps), and the resulting list is used to create a secondary index. Each extracted value becomes one SI entry.
- **Pre-filtered input:** A `MapExp.getByKeyList` narrows a map to specific keys before `CdtExp.selectByPath` applies further path filtering. This was the 8.1.1 workaround for IN-list key selection before `mapKeysIn` was introduced in 8.1.2.

It takes the same context vocabulary as the direct operations, including `mapKeysIn` and `andFilter` (8.1.2+).

**Example — filter expression with CdtExp.selectByPath (Java):**

```java
// Extract all license plates from the vehicles list-of-maps
// then check if target plate exists
QueryPolicy policy = client.copyQueryPolicyDefault();
policy.filterExp = Exp.build(
    ListExp.getByValue(ListReturnType.EXISTS,
        Exp.val(targetPlate),
        CdtExp.selectByPath(Exp.Type.LIST, SelectFlags.VALUE,
            Exp.listBin("vehicles"),
            CTX.allChildren(),
            CTX.mapKey(Value.get("license")))));
```

### Expression index creation with path expressions

Path expressions enable creating secondary indexes on values extracted from nested CDT structures. The pattern:

1. Define a `CdtExp.selectByPath` expression that extracts the target values into a list.
2. Create an SI with `IndexCollectionType.LIST` so each extracted value is indexed individually.
3. Query using the index name or the same expression as a filter.

**Example — index all license plates from a list-of-maps (Java):**

```java
// Step 1: define the expression
Expression licensesExp = Exp.build(
    CdtExp.selectByPath(Exp.Type.LIST, SelectFlags.VALUE,
        Exp.listBin("vehicles"),
        CTX.allChildren(),
        CTX.mapKey(Value.get("license"))));

// Step 2: create the secondary index
IndexTask task = client.createIndex(null, "test", "demo",
    "idx_vehicle_license", IndexType.STRING,
    IndexCollectionType.LIST, licensesExp);
task.waitTillComplete();

// Step 3a: query by expression
Statement stmt = new Statement();
stmt.setFilter(Filter.contains(licensesExp,
    IndexCollectionType.LIST, "7XYZ789"));

// Step 3b: alternatively, query by index name (simpler production pattern)
stmt.setFilter(Filter.containsByIndex("idx_vehicle_license",
    IndexCollectionType.LIST, "7XYZ789"));
```

Querying by index name avoids rebuilding the expression on the client side and is the simpler production pattern. The same `selectByPath` / `allChildrenWithFilter` operations can be used as operation projections on the query result to return only matching elements rather than the full record.

---

## SelectFlags (return mode)

Control what the server returns from `selectByPath`:

| Flag                   | Description                                                                                                                                         |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **MATCHING_TREE**      | Return subtree from bin to leaves, including only nodes that passed filters. Preserves parent-child hierarchy. Good for "return filtered document." |
| **VALUE / VALUES**     | Return a flat list of the **values** of the nodes finally selected.                                                                                 |
| **MAP_KEY / MAP_KEYS** | For final nodes that are map elements, return only the **map keys** (e.g. list of SKU ids).                                                         |
| **MAP_KEY_VALUE**      | Return a list of key-value pairs for final selected map elements.                                                                                   |
| **NO_FAIL**            | On type mismatch (e.g. expecting map, got string), skip that element instead of failing the whole operation.                                        |

Without **NO_FAIL**, any type mismatch aborts the operation with no partial results. `NO_FAIL` is useful when matching across heterogeneous collections where not every element has the expected data type (e.g. a catalog where some product variants are maps and others are lists).

---

## Example: list of structs (e.g. user's vehicles)

Without path expressions, iterating over a list of map elements (e.g. vehicles as structs with make, color, license) to filter or index by a field is not supported; the only option is to **denormalize** (e.g. a separate bin holding a list of license strings) to index or query by license. Path expressions allow a single **list of maps** in one bin (e.g. `vehicles`: `[{make, color, license}, ...]`) and:

- **Filter in place:** Use `selectByPath` with a context that iterates over the list's children and a filter on `license` (e.g. `MapExp.getByKey(..., "license", Exp.mapLoopVar(LoopVarPart.VALUE)) == "ABC123"`) to return only vehicles (or only matching vehicle structs) where license equals a given string.
- **Index by field:** Create an **expression index** that iterates over the vehicle map structures inside the list and indexes the `license` (or other) field — so you can query "all users with license plate X" without a separate bin holding a list of license strings.

This **list-of-maps** pattern (list of structs with named fields) is directly useful for a canonical example application: e.g. one record per user, one bin `vehicles` = list of `{make, color, license}`, with server-side filter by license or expression index on license. See [concepts-and-patterns.md](concepts-and-patterns.md) for how this fits with other modeling choices. For applied patterns and the broader document-modeling paradigm built on this concept, see [Document modeling with path expressions](#document-modeling-with-path-expressions) below.

---

## Document modeling with path expressions

### Conceptual frame: path expressions as document modeling

Path expressions enable a **document-modeling paradigm** for Aerospike CDTs: storing structured documents (lists of maps with named fields, or nested map hierarchies) inside CDT bins and using path expressions for server-side field-level query and mutation. Fields are addressed by name, not position — eliminating the positional trade-offs inherent in baseline CDT patterns (e.g., the element-0 gate in ordered lists of tuples).

**Analogy: JSONPath for MessagePack.** Path expressions relate to Aerospike CDTs much as JSONPath relates to JSON documents. Both provide declarative traversal, field-level filtering, and projection over nested structures, executed server-side. The key differences: Aerospike path expressions operate on MessagePack-encoded CDTs with typed loop variables and support **in-place modification** (`modify_by_path`), not just read-only query. This makes them suitable for write-heavy workloads where field-level mutations (mark-read, toggle visibility, increment counters) must happen atomically on the server without read-modify-write round trips.

**When to reach for document modeling:**

- The modeling requirement includes **field-level server-side filtering** (e.g., "return only unread notifications," "return only items of type X") that baseline CDT operations cannot serve because the target field is not at a leading position.
- **Multi-field mutations** are required on elements within a collection (e.g., mark-read + block-actor + type-filtered query on the same event timeline), and the element-0 gate produces irreconcilable trade-offs between dedup, time-ordering, and mutation targeting.
- The mutation-compatibility verification in the [modeling checklist](new-app-modeling-checklist.md) § 5.7 surfaces multiple gaps that would otherwise require client-side workarounds (read-filter-write, set-difference logic).

**Version gate:** Requires Aerospike DB 8.1.2+. Always maintain a baseline CDT fallback (ordered list of tuples or map-keyed variant) for environments where path expressions are unavailable.

### Data modeling implications

- **Document-style records:** One record (e.g. catalog) can hold a large map of entities (e.g. productId → product). Path expressions let you "query" that document server-side: e.g. "featured products with at least one in-stock variant," returning only those products and only in-stock variants.
- **List of structs:** A bin can hold a **list of maps** (each map = one "struct" with fields like make, color, license). Path expressions let you iterate over that list and filter or project by field; expression indexes let you index a field across all list elements — no need for a separate denormalized bin (e.g. list of license strings) just to query or index.
- **Avoid denormalization:** You can keep a single source of truth (one record per catalog or user) and use path expressions to project subsets (e.g. by segment, by time window, by flag) instead of maintaining separate flattened records.
- **IN-style membership:** There is no cluster-level SQL-style `IN`. Within a **single record**, DB 8.1.2+ provides `CTX.mapKeysIn(keys...)` for native IN-list key selection in path expressions. On 8.1.1, the workaround is `allChildrenWithFilter` with `ListExp.getByValue(COUNT, Exp.stringLoopVar(MAP_KEY), Exp.val(roomIds)) > 0` to keep only entries whose map key is in `roomIds`. See [Performance](#key-selection-and-combined-filtering-812) for a comparison of the three approaches.
- **Combining filters:** Multiple conditions (e.g. quantity > 0 AND price < 50) use `Exp.and(...)` / `Exp.or(...)` with each condition reading from the same loop variable (e.g. variant map).
- **Return shape:** Use MATCHING_TREE for "filtered document," MAP_KEY for "list of ids," VALUE for "list of values" — keeps payloads small when you only need keys or a flat list.

### Applied pattern: list-of-structs event timeline

This pattern applies the list-of-maps (list-of-structs) concept to **day-bucketed event timelines** such as notifications, activity feeds, or audit logs. It resolves the field-level filtering and modification limitations of both the ordered-list-of-tuples and map-keyed Pack 4 variants (see [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 5.7).

**Experimentally validated** against Aerospike DB 8.1.1.1 — all operations below were tested and confirmed against the 8.1.1 preview. The 8.1.2 production release adds `mapKeysIn` and `andFilter` context types but does not change the operations used in this pattern.

#### Structure

```
key: notif:{user_handle}:{YYYY-MM-DD}
bin "items": unordered list of maps
  [
    {id: "like:alice:tweet_42", type: "like", actor: "alice", target: "tweet_42", read: false, visible: true, ts: 1711100000000},
    {id: "rt:bob:tweet_42", type: "retweet", actor: "bob", target: "tweet_42", read: false, visible: true, ts: 1711100500000},
    ...
  ]
```

Each element is a map with named fields. The list is **unordered** (append-only); insertion order provides chronological ordering.

#### Operations

| Operation                        | Mechanism                                                                                                                      | Baseline CDT needed?  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| **Write**                        | `list_append`                                                                                                                  | Yes (all versions)    |
| **Paginate (newest N)**          | `list_get_by_index_range(-N, N)`                                                                                               | Yes (all versions)    |
| **Trim (oldest N)**              | `list_remove_by_index_range(0, N)`                                                                                             | Yes (all versions)    |
| **Mark-read**                    | `modify_by_path` with filter `read == false`, expr `MapPut("read", true)`                                                      | No — path expressions |
| **Unread filter**                | `select_by_path` with filter `read == false`, flags `SELECT_VALUE`                                                             | No — path expressions |
| **Block-actor (mark-invisible)** | `modify_by_path` with filter `actor == blocked_user`, expr `MapPut("visible", false)`                                          | No — path expressions |
| **Visible-only read**            | `select_by_path` with filter `visible == true`, flags `SELECT_VALUE`                                                           | No — path expressions |
| **Type-filtered query**          | `select_by_path` with filter `type == "like"`, flags `SELECT_VALUE`                                                            | No — path expressions |
| **Remove-by-predicate**          | `modify_by_path` with filter (e.g. `actor == blocked_user`), expr `ResultRemove`                                               | No — path expressions |
| **Exact-duplicate dedup**        | `list_append` with `ADD_UNIQUE, NO_FAIL` — server K-orders maps before comparison; `KeyOrderedDict` is defensive, not required | Yes (all versions)    |
| **Identity-based dedup**         | `select_by_path` check for existing `id` field value, then conditional append                                                  | No — path expressions |

#### Key design properties

**Named fields eliminate the element-0 gate problem.** With ordered lists of tuples, the leading element determines which CDT operations work selectively — placing identity at element 0 enables dedup but prevents time-based filtering, and vice versa. With list-of-structs, every field is addressable by name via path expressions. There is no positional trade-off.

**Mark-invisible over remove for index stability.** Removing list elements shifts subsequent indexes, breaking client-side "last seen index" tracking. Setting `visible: false` via `modify_by_path` preserves list length and index positions. The read path filters on `visible == true` using the same `select_by_path` mechanism. Invisible entries expire with the bucket's record-level TTL. Note: path expressions do support server-side removal of matched elements via `ResultRemove` with `modify_by_path` (see [Operations](#modifybypath-write) above and the Remove-by-predicate row in the table). The mark-invisible preference here is a design choice for index stability in the event-timeline pattern, not a platform limitation.

**Storage overhead.** Map key strings add ~58 bytes/element overhead vs equivalent positional tuples (measured with 7-field maps). At 50 elements/day: ~2.9 KiB extra (negligible). At 500 elements/day: ~29 KiB (include in sizing math via `avg_event_payload_bytes`).

#### When to use this pattern vs baseline Pack 4

| Condition                                                                                                      | Recommended structure                                                                                             |
| -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Paginate-then-mark consumption model, mutations are batch-only post-display                                    | Ordered list of tuples (baseline)                                                                                 |
| Key-targeted mutations (dedup by natural key, per-event update), all DB versions                               | Map-keyed variant (baseline)                                                                                      |
| Server-filtered-unread, type-filtered queries, block-actor side effects, or multi-field mutations; DB >= 8.1.2 | **List-of-structs (this pattern)**                                                                                |
| DB < 8.1.2 but need field-level filtering                                                                      | Fall back to ordered list or map-keyed with client-side filtering, or one-event-per-record with expression filter |

#### Relation to Pack 4 in the modeling checklist

The three structures form an escalation, not peer alternatives:

```
ordered list of tuples (baseline CDT, all versions)
  → map-keyed variant (baseline CDT, all versions, trades time-order for key-access)
    → list-of-structs (DB 8.1.2+, trades version dependency for field-level access)
```

This pattern is documented here (in the path-expressions reference) rather than as a third Pack 4 variant in the [modeling checklist](new-app-modeling-checklist.md) to prevent agents from selecting a version-gated pattern without checking the gate. The checklist's server-filtered-unread verdict and element-0 cross-check include forward references to this section.

---

## Advanced usage (high level)

- **Loop vars for keys:** Filter by map key (e.g. productId) using `Exp.stringLoopVar(LoopVarPart.MAP_KEY)` in a regex or comparison (e.g. keys starting with `"10000"`).
- **Alternative return modes:** Same context stack, different selectFlags — e.g. MAP_KEY to get only in-stock variant SKUs for featured products.
- **modifyByPath:** One operation to update many nested elements (e.g. increment quantity by 10 for all in-stock variants of featured products) using an expression that reads from `Exp.mapLoopVar(VALUE)` and writes back with `MapExp.put`.
- **Element removal via `ResultRemove`:** Instead of a value-computing expression, pass `ResultRemove` as the modify expression. The server removes every element reached by the context stack. Combined with `allChildrenWithFilter`, this provides selective deletion of nested elements by predicate (e.g., remove all entries where `status == "expired"` from a map of maps). See [Python client path expressions tests](https://github.com/aerospike/aerospike-client-python/blob/19.1.0/test/new_tests/test_path_expressions.py#L562-L586) for a verified example.
- **Chained filters:** AND/OR over expressions that all refer to the current element via the same loop variable.

---

## Limits and performance

- **Nesting depth:** Up to **64 levels**. Database 8.2.0 and later limit List and Map nesting, and the number of context levels in a path, to 64.
- **Elements:** No hard limit on number of elements, but very large CDTs (e.g. millions of elements) can increase latency; consider partitioning across records or using secondary indexes to narrow scope before path expressions.
- **Performance factors:** Result size (MATCHING_TREE vs MAP_KEY), filter complexity, nesting depth, CDT size. Prefer server-side path filtering over full-record fetch + client filter when possible; benchmark with realistic data.

### Batch inlining with large records

Path expression workloads typically involve nested CDT documents well above 1 KiB per record. When batching these operations, the default batch policy `allowInline=true` serializes the entire sub-batch on a single service thread for in-memory namespaces. For records larger than about 1 KiB, this is slower than non-inlined execution, which distributes the sub-batch across multiple service threads.

**Guidance:** Set `allowInline` to `false` (or the language-equivalent batch policy flag) when batching path expression operations against an in-memory namespace whose records exceed about 1 KiB. For SSD-based namespaces the default `allowInlineSSD=false` already disables inlining.

### Troubleshooting

Common error scenarios and solutions:

| Error                          | Cause                                                   | Solution                                                                                                          |
| ------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OP_NOT_APPLICABLE`            | Target bin is not a map or list                         | Verify bin type before applying path expressions, or use a record-level filter expression to skip non-CDT records |
| `PARAMETER_ERROR`              | Malformed filter expression or invalid type reference   | Check expression syntax; ensure `LoopVarPart` matches container type (`MAP_KEY` for maps, `LIST_INDEX` for lists) |
| Type mismatch during traversal | Filter expects different type than actual data          | Use `NO_FAIL` to skip invalid elements, or validate data schema                                                   |
| Operation timeout              | CDT too large or deeply nested                          | Increase client timeout, reduce data size, or restructure to reduce nesting depth                                 |
| Empty result when data exists  | Filters too restrictive or path doesn't match structure | Test incrementally: start with `allChildren()` without filters, then add filters one at a time                    |

---

## Relation to other research

- **CDT API ([cdt-api.md](cdt-api.md)):** Path expressions use the same underlying List/Map types and ordering; context steps are analogous to nested CDT context (BY_MAP_KEY, BY_LIST_INDEX, etc.), but with expression-based filtering and loop variables.
- **Data modeling ([concepts-and-patterns.md](concepts-and-patterns.md)):** Path expressions are a good fit for "one record, one big map/list" patterns (e.g. user profile store, product catalog) when you need server-side filtering, projection, or bulk updates on nested subsets without denormalizing into many records or bins. The **list-of-structs** pattern (e.g. user's vehicles as list of maps with make, color, license) is called out there for a canonical example application: one bin, path expressions to filter by field, expression index to query by field, no separate denormalized bin needed.
- **Modeling checklist Pack 4 ([new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 5.7):** The [list-of-structs event timeline applied pattern](#applied-pattern-list-of-structs-event-timeline) above resolves the server-filtered-unread and mark-read mechanism gaps in the Pack 4 map-keyed storage variant. The checklist's mutation-compatibility fallback list, server-filtered-unread verdict, and element-0 cross-check reference document modeling as a named resolution path, linking to this section.
