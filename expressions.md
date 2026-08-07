# Aerospike Expressions — Research Summary

**Summary:** Aerospike Expressions is a domain-specific language for comparing and manipulating record bins and metadata. It is strictly typed, functional, and intentionally **non–Turing complete** (no iteration or recursion), which keeps execution fast and predictable. Expressions are used for: **filtering records** (WHERE clause for scans and queries), **controlling operations** (e.g. whether a write proceeds), **XDR filtering** (what replicates to remote DCs), and **extending transactions** (computed bins, atomic cross-bin operations). This note summarizes content from an Aerospike Summit talk and the official [Expressions](https://aerospike.com/docs/develop/expressions/) docs.

**Status:** Research. Use official [Expressions documentation](https://aerospike.com/docs/develop/expressions/) for current API and versioning (latest as of this writing: Aerospike 8.1.2, GA 2026-04-16). Under that section the docs include: Overview, [Nested CDTs](https://aerospike.com/docs/develop/expressions/nesting) (comprehensive tutorial covering CDT operations, expression composition, and path expressions together), Path expressions (see [path-expressions.md](path-expressions.md)), Declare and control, Comparison, Logic, Arithmetic, Record metadata, Record storage, List bin ops, Map bin ops, Blob/bitwise bin ops, HyperLogLog bin ops. Use those pages for operator and function reference.

**Scope and assumptions for new app modeling:** This is an advanced feature reference, not a standalone modeling checklist. Start with [new-app-modeling-checklist.md](new-app-modeling-checklist.md), verify DB/client version support for expression features you intend to use, and define fallbacks before implementation.

**Source:** Summit 2021 talk ["Unleashing the Power of Expressions"](https://www.youtube.com/watch?v=ebRLnXvpWaI) (transcript) and companion code: [aerospike-examples/aerospike-expressions-workshop](https://github.com/aerospike-examples/aerospike-expressions-workshop) (Python; scripts below).

---

## Workshop code patterns (aerospike-expressions-workshop)

Concrete patterns from the repo. Python client: `aerospike_helpers.expressions` as `exp`, `aerospike_helpers.operations.expression_operations` as `opexp`, `aerospike_helpers.expressions.arithmetic` for `Mod`, `Sub`, `Abs`, `Add`.

### scan-filter.py — Filter expressions on scan and background touch

- **Namespace/set:** Configurable; set `scan-filter` for the demo. Records: integer key, bin `num` (same as key), even keys TTL 1h, odd TTL 1m.
- **Filter on metadata:** `exp.GT(exp.SinceUpdateTime(), 1000).compile()` — records updated more than 1000 ms ago. Passed as `policy = {"expressions": ...}` to `client.scan(...).foreach(..., policy)` or to background scan. **SinceUpdateTime** is in milliseconds.
- **Filter on metadata + bin:** `exp.And(exp.LT(exp.IntBin("num"), 10), exp.Eq(arithmetic.Mod(exp.IntBin("num"), 2), 1), exp.LT(exp.TTL(), 60*30))` — odd `num` &lt; 10 and TTL &lt; 30 minutes. **TTL()** in seconds.
- **Filter on key:** `exp.KeyInt()` for integer user key; e.g. `exp.GT(exp.KeyInt(), 10)` and `arithmetic.Mod(exp.KeyInt(), 2)` for odd keys &gt; 10.
- **Background scan with op:** `scan.add_ops([operations.touch()])`, `scan.execute_background(policy)` to extend TTL only on records that pass the filter. `nobins: True` when only keys/metadata are needed.
- **Note:** Namespace must allow TTL (`allow-ttl-without-nsup`, `default-ttl`) or puts with TTL can return `AEROSPIKE_ERR_FAIL_FORBIDDEN`.

### messages.py — Filter expressions on get and get_many

- **Data model:** Set `messages`. Records keyed by message ID (int). Bins: `msg` (string), `access` (list of usernames). Public = empty `access`; personal = one username (e.g. `["cher"]`); private = two or more (e.g. `["cher", "dee"]`).
- **Single-record get with filter:** `is_public = exp.Eq(exp.ListSize(None, exp.ListBin("access")), 0).compile()`; `client.get(key, policy={"expressions": is_public})`. If record exists but filter fails → **exception.FilteredOut** (e.g. code 27).
- **Batch get with filter:** `client.get_many(keys, policy={"expressions": is_public})` returns only records that pass the filter.
- **Let for reuse:** `exp.Let(exp.Def("access_size", exp.ListSize(None, exp.ListBin("access"))), exp.And(exp.Eq(exp.Var("access_size"), 1), exp.Eq(exp.ListGetByIndex(None, aerospike.LIST_RETURN_VALUE, exp.ResultType.STRING, 0, exp.ListBin("access")), "cher"))).compile()` — personal message for "cher" (list size 1 and first element "cher"). Alternative without Let: same condition inlined (list size and ListGetByIndex in one expression).
- **List ops in filter:** `exp.ListSize(None, exp.ListBin("access"))`, `exp.ListGetByIndex(..., 0, "access")`, `exp.ListGetByValue(None, aerospike.LIST_RETURN_COUNT, "cher", exp.ListBin("access"))` (count of "cher" = 1 for “cher is in access”).
- **Metadata in filter:** `exp.SinceUpdateTime()` (ms) e.g. for “private messages &lt; 5 seconds old”; combine with list conditions. If metadata fails first, storage is not read (two-phase).

### simple-op-exp.py — Read and write operation expressions

- **Read metadata as bins:** `since_update = exp.SinceUpdateTime().compile()`, `get_lut = exp.LastUpdateTime().compile()`. Ops: `oh.read("us")`, `opexp.expression_read("since_update", since_update)`, `opexp.expression_read("lut", get_lut)`. Returns numeric values under chosen bin names (since_update in ms; lut in nanoseconds since epoch).
- **Cross-bin write expression:** `calc` = `exp.Let(exp.Def("combined", exp.ListAppendItems(None, None, exp.ListBin("b"), exp.ListBin("a"))), exp.Def("combined_min", exp.ListGetByRank(..., 0, exp.Var("combined"))), exp.Def("combined_max", exp.ListGetByRank(..., -1, exp.Var("combined"))), exar.Sub(exp.Var("combined_max"), exp.Var("combined_min"))).compile()` — append list `a` to `b`, take min and max by rank, return max − min. `opexp.expression_write("e", calc)` writes that value to bin `e`.
- **Read expression using bin created in same transaction:** `get_min = exp.Min(exp.IntBin("c"), exp.IntBin("d"), exp.IntBin("e")).compile()`; first op writes `e`, then `opexp.expression_read("min", get_min)` reads min(c,d,e). Same transaction so `e` is visible.
- **Arithmetic:** `exar.Sub`, `exp.Add` (e.g. `exp.Add(exp.IntBin("e"), exp.IntBin("d")).compile()` returned as `expression_read("sum_d_e", ...)`).

### exp-composition.py — Composing list operations in expressions

- **Unordered “slice” of value range:** `exp.ListGetByIndexRange(None, aerospike.LIST_RETURN_VALUE, 0, 3, exp.ListGetByValueRange(None, aerospike.LIST_RETURN_VALUE, 10, 90, exp.ListBin("n")))` — elements of `n` between 10 and 90, then first four (index 0–3). Order is unspecified.
- **Ordered slice:** Same value range passed to `exp.ListSort(None, aerospike.LIST_SORT_DEFAULT, ...)` then `ListGetByIndexRange(..., 0, 3, ...)` — sort then take first four. Read-only; no change to stored bin.
- **Return as synthetic bins:** `opexp.expression_read("unordered0-3", unordered_slice)`, `opexp.expression_read("ordered-03", ordered_slice)` so the operate result contains these under custom names; record bins are unchanged.

### roster.py — Churn with Cond and multi-op transaction

- **Data model:** Single record (e.g. key `"sc-sharks-02-blue"`). Bin `roster`: ordered list of player names (unique list). Bins `tmp` (temp size), `churn` (cumulative churn count).
- **List policy:** `LIST_WRITE_ADD_UNIQUE | LIST_WRITE_PARTIAL | LIST_WRITE_NO_FAIL`, `LIST_ORDERED` — merge new players, ignore duplicates, no fail on duplicate.
- **Current size expression (write to tmp):** `current = exp.Cond(exp.Eq(exp.BinType("roster"), aerospike.AS_BYTES_LIST), exp.ListSize(None, exp.ListBin("roster")), 0).compile()` — if `roster` is a list, return its size else 0. Written with `opexp.expression_write("tmp", current)`.
- **Churn expression:** `churn` = previous churn (if bin exists and is integer) plus absolute size change. Uses `exp.IntBin("tmp")` (size before changes), `exar.Abs(exar.Sub(exp.ListSize(None, exp.ListBin("roster")), exp.IntBin("tmp")))` for delta; `exp.Cond(exp.Eq(exp.BinType("churn"), aerospike.AS_BYTES_INTEGER), exp.IntBin("churn"), 0)` for prior churn; `exp.Add(prior_churn, delta)` written with `expression_write("churn", churn)` so churn accumulates across updates.
- **Transaction pattern:** `expression_write("tmp", current)` → `list_append_items("roster", add_2017, list_policy)` → `expression_write("churn", churn)` → `expression_write("tmp", current)` (refresh size) → `list_remove_by_value_list("roster", remove_2017, ...)` → `expression_write("churn", churn)` → `oh.write("tmp", aerospike.null())` (clear temp) → `read("churn")`, `read("roster")`. All in one `client.operate(key, ops)`.

---

## What are expressions

- An **expression** is a syntactic entity that **evaluates to a value** (as opposed to a statement, which carries out instructions and does not return a value). Expressions can contain variables, operators, and functions and return a primitive (string, integer) or a complex value (e.g. list of integers).
- **Aerospike Expressions** is strongly typed, functional, and Lisp-influenced. Syntax uses **Polish (prefix) notation**: operator then operands (e.g. `+ 4 3`, `and (eq a 1) (gt b 2)`).
- **Immutability:** Bin data and record metadata accessed inside an expression are **immutable**. Changes happen on an ephemeral copy; when reading a bin you get its value at the start of the operation. The copy is released when each sub-expression completes, which defines the scope of sub-expressions. If an expression modifies a bin, those changes apply to a temporary copy; only the final result of an expression **write** is stored to the target bin.
- **Type system (reference):** The official docs divide expression types into **value** vs **bin** and use parameter names `t_value`, `t_bin_expr`, `t_expr` (and `library_specific` for client-language types). Value subtypes include integer, float, string, blob, list, map, geojson, hll, boolean, nil, and AUTO (type inference in some clients). See [Expressions — Syntax and behavior](https://aerospike.com/docs/develop/expressions/) and the type/operator pages for full reference.

---

## Where expressions are used

| Feature | Introduced | Purpose |
|--------|------------|---------|
| **Record filter expressions** | 5.2 | Select records based on a boolean expression (WHERE clause for scans, queries, batch reads). |
| **XDR filter expressions** | 5.3 | Filter which records replicate to specific XDR destinations (namespace + destination). |
| **Operation expressions** | 5.6 | Extend read/write bin operations with computed values; atomic cross-bin operations; also used to **control** whether an operation (e.g. write) proceeds. |
| **Secondary index expressions** | 8.1 | Index the **computed value** of an expression instead of a raw bin — more memory-efficient secondary indexes on large data sets. Create via asadm; queries use the expression index by matching the expression or by index name. |

### Secondary index expressions (expression indexes)

Expression indexes (Database 8.1+) build **sparse** secondary indexes: the index expression is evaluated per record, and **only records for which the expression returns a value of the index’s declared type are included**. Records that don’t match are omitted from the index, reducing memory and improving query cost.

**Skipping indexing for some values:** Use **`cond`** with **`unknown()`** as the default. When the condition is true, return the value to index (e.g. a bin or computed value); when false, return **`unknown()`** so the record is **excluded** from the index. Example (index age only for adults in certain countries): `cond(and(ge(intBin("age"), 18), countryInTargetList), intBin("age"), unknown())` — adults in the list get their age indexed; everyone else is not indexed. See the [Tutorial for Expression Indexes](https://aerospike.com/docs/database/learn/tutorials/quick-start-expression-index/) for full client examples (Java, Python, C#, C, Node.js) and asadm usage.

**When to use:** Use expression indexes when you would otherwise persist a derived bin solely to make it indexable, index every record although only a subset is needed for queries, or re-implement the same filter logic in multiple clients. Benefits: simpler application code, smaller index footprint, lower infrastructure cost, faster queries.

**Creation:** Build the expression in the client (same expression API as filters/operations); obtain the Base64-encoded expression (e.g. `get_expression_base64` / `GetBase64`); create the index via asadm with `exp_base64` and the index type (numeric, string, etc.). Index metadata is defined by the value type (ktype) and index type (itype: none, list, map-keys, map-values). On write, if the expression fails to produce a value of the declared type, no SI entry is added (same idea as “bin missing or wrong type” for bin-based SIs).

**Expression index with path expressions.** Path expressions can define the indexed value for a secondary index expression. The canonical pattern: use `selectByPath` inside an expression to extract values from nested structures (e.g. all license plates from a list-of-maps), then create an SI on the resulting list. Each element of the extracted list becomes a separate index entry (collection type `LIST`). Example: `CdtExp.selectByPath(Exp.Type.LIST, SelectFlags.VALUE, Exp.listBin("vehicles"), CTX.allChildren(), CTX.mapKey("license"))` extracts all license values; creating an SI with `IndexCollectionType.LIST` on this expression indexes each license plate individually. See the [vehicles worked example](https://aerospike.com/docs/develop/expressions/nesting#create-an-expression-index-on-license-plates) for full multi-language code.

**Query by index name.** Once an expression index exists, queries can reference it by name instead of passing the expression again. This avoids rebuilding the expression on the client side and is the simpler production pattern. Example: `Filter.containsByIndex("idx_vehicle_license", IndexCollectionType.LIST, "7XYZ789")` (Java). The same operation projection (e.g. `selectByPath` for filtering matching elements, bin reads) applies to both expression-based and name-based query filters. See the [query by index name example](https://aerospike.com/docs/develop/expressions/nesting#alternatively-query-by-index-name).

---

## Filter expressions

- **Purpose:** Select records based on whether a **boolean expression** evaluates to true. Act as the **WHERE clause** for scans and queries.
- **Commands that support (record) filters:** read, write, delete, record UDFs, transactions, batched commands, primary index queries (scans), secondary index queries. Filters also apply to background operations (e.g. background scan with touch).
- **Supported functions (official docs):** Metadata functions; full List and Map APIs including nested context; bitwise functions for blobs; GeoJSON geo-spatial; HyperLogLog (HLL). See [cdt-api.md](cdt-api.md) for CDT ops; expression docs for metadata/comparison/logic/arithmetic.
- **Important:** Filter expressions run **only when the record exists**. They do not execute if a read fails to find the record or if a write is creating a new record. The filter does not provide “if record missing then …” behavior.

### Guarding writes on record existence

A filter expression cannot gate "record must already exist" — the filter does not run when the record is missing, so writes to a missing record (including `operate` with any write op) bypass the filter and, for create-friendly op lists, create the record. To veto a write on a missing record, combine the filter with **`WritePolicy.recordExistsAction = UPDATE_ONLY`** (Python: `policy={"exists": aerospike.POLICY_EXISTS_UPDATE}`):

```python
client.operate(
    key,
    ops,
    policy={
        "expressions": filter_exp,
        "exists": aerospike.POLICY_EXISTS_UPDATE,
    },
)
```

On a missing record the policy layer rejects before the op list executes; the client raises `RecordNotFound` (code 2). On an existing record the filter is evaluated normally; a filter reject raises `FilteredOut` (code 27). Handle the two as distinct `except` clauses at the call site.

This is Aerospike's officially-documented pattern for the problem — see [Use Write Policies To Replace Existence Checks](https://aerospike.com/docs/develop/tutorials/intro/intro_to_transactions/operate_applies_one_or_more_operations) in the intro-to-transactions tutorial. It composes cleanly with filter expressions: the filter keeps the "record exists, additional predicate must hold" condition; the policy flag adds the "record must exist" condition that filters cannot express.
- **Scan vs query (terminology):** A **scan** uses the primary index to traverse all records in the namespace or set (no secondary index). A **query** is when a **secondary index** is used to index by some bin (or expression). In both cases, a **filter expression** is the WHERE clause. **Set indexes** (5.6): automatic indexing of set membership so “all records in set S” in a large namespace can be found efficiently without a full primary-index scan.
- **Two-phase execution:** For performance, filter expressions use a **metadata phase** then optionally a **storage phase**. See [Expression execution model](#expression-execution-model) below.
- **Examples (from talk):** Filter by last-update time (since_update_time &gt; 1000 ms); filter by bin value (e.g. list size of `access` = 0 for “public message”) and metadata (TTL &lt; 30 minutes); batch read with filter for “public messages” or “personal messages” (list size 1 and index 0 = username). When a record is filtered out on a single-record read, the client receives an explicit “filtered out” result (e.g. error code 27), distinct from “record not found.”

---

## Operation expressions

- **Purpose:** Extend **read** and **write** bin operations with **computed values** from record metadata and bin data. Enable **atomic cross-bin operations** that previously required UDFs (e.g. read bin A, compute from it, write to bin B in one transaction).
- **Used in:** The **operate** command (single-record multi-op transactions) and **background scan** operations. Applied **after** any filter expression (if present); if the filter passes, the operation expression runs as part of the transaction.
- **Read expressions:** Return computed values (e.g. record metadata such as last update time or since-update time, or results of list/map operations) and can be returned under a chosen bin name (like a SQL column alias).
- **Write expressions:** Write the result of an expression into a bin (e.g. “min of bins c, d, e” written to bin `e`). Operations in one transaction see the ephemeral copy; e.g. first op writes expression result to bin `e`, later ops can read `e`.
- **Composition:** Expressions can **compose** existing CDT and metadata operations. Example: get list bin → pass to get_by_value_range(10, 90) → pass to get_by_index_range(0, 3) to implement a “slice” over a value range; or sort the value-range result then take first N for an “ordered slice.” This creates new behavior without new server ops.
- **Let construct:** Declare variables (name, value pairs) and a final expression that can reference them via `var(name)`. Use when the same (possibly expensive) sub-expression would otherwise be repeated; variables are in scope for the final expression only.
- **Cond (conditional):** Switch-like: a series of (condition, action) pairs. First condition that evaluates to true has its action evaluated and returned; rest are skipped. A final default action is evaluated if no condition matches. Default can be **unknown** to abort the operation (error code 26, “not applicable”); in **filter** expressions, prefer default **false** so the metadata phase can short-circuit without going to storage. In **operation** expressions, unknown can be used; policies (e.g. no-fail for read/write expressions) can suppress abort in a transaction.
- **Example (roster/churn):** Compute “churn” (e.g. absolute difference in roster size before and after list operations). Use a read expression to get current list size and write it to a temporary bin; run list append/remove with unique/ordered/partial/no-fail policy; compute churn expression (abs(size_after − size_before)); optionally accumulate churn in another bin. All in one atomic operate transaction.

---

## Declare and control-flow (5.6.0+)

Variables and branching for reuse and conditional logic. All introduced in 5.6.0. See [Declare and control-flow](https://aerospike.com/docs/develop/expressions/declare) for full API.

| Op | Purpose |
|----|--------|
| **let**(def(...), def(...), ..., expr) | Define variables in scope; returns the last expression. Use to reuse an expensive sub-expression (e.g. HLL count) in the final expr. |
| **def**(name, value) | Define a variable inside a `let`. |
| **var**(name) | Reference a variable defined in the same `let`. |
| **cond**(condition0, action0, condition1, action1, ..., default-action) | Test/action pairs; first condition that is true returns that action; no further tests evaluated. Default is used if all tests are false. All actions must be the same type or `unknown`. |
| **unknown**() | Returns the trilean "unknown". In an **operation expression**, this aborts the op with error 26 (not applicable); use in `cond` default to skip a write (e.g. "add 1 to count only if count &lt; 10"). **NO_EVAL_FAIL** policy allows the transaction to continue. **Avoid in filter expressions** — unknown can force the storage phase. Same value as failed expressions (e.g. division by zero). |

**Example (cond):** Letter grade from numeric bin: `cond(ge(grade, 90), "A", ge(grade, 80), "B", ..., "F")`. **Example (let):** Filter where HLL bin "cookies" count is &lt; 1k or &gt; 100M: `let(def("count", hllGetCount("cookies")), or(lt(var("count"), 1000), gt(var("count"), 100000000)))`.

---

## Comparison expressions (5.2.0+)

Boolean comparison ops for filters and cond tests. Left and right must be the same fundamental type unless noted. See [Comparison](https://aerospike.com/docs/develop/expressions/comparison) for full API and examples.

| Op | Purpose |
|----|--------|
| **eq**(left, right), **ne**(left, right) | Equal / not equal. For maps: `eq` of unordered vs ordered map (or same elements, different order) can be false. |
| **lt**, **le**, **gt**, **ge**(left, right) | Less than, less-or-equal, greater than, greater-or-equal. |
| **cmp_regex**(regex_string, options, string_expr) | True if regex matches the string (e.g. bin or nested value). Options: flags (e.g. ICASE, NEWLINE). |
| **cmp_geo**(left, right) | True if left GeoJSON is contained in or contains right (geojson_expr). |

**Examples:** `eq(stringBin("fname"), val("Frank"))`; `ge(intBin("age"), val(21))`; `gt(ttl(), val(365*24*3600))` for TTL &gt; 1 year; `cmp_regex("^555.*", 0, stringBin("phone_num"))` for area code; range: `and(ge(stringBin("lname"), val("o")), lt(stringBin("lname"), val("p")))`.

---

## Logic expressions (5.2.0+; exclusive 5.6.0+)

Logical operators combine boolean expressions and return a boolean. Used in filters and in `cond` tests. See [Logic](https://aerospike.com/docs/develop/expressions/logic) for full API and C/Java examples.

| Op | Purpose |
|----|--------|
| **and**(arg0, arg1, ...) | Returns `false` if any boolean_expr is false; otherwise `true`. |
| **or**(arg0, arg1, ...) | Returns `true` if any boolean_expr is true; otherwise `false`. |
| **not**(arg) | Returns `true` if the boolean_expr is false; otherwise `false`. |
| **exclusive**(arg0, arg1, ...) | Returns `true` if **exactly one** boolean_expr is true (mutually exclusive); otherwise `false`. Introduced 5.6.0. |

**Examples:** Key exists and key &lt; 1000: `and(keyExists(), lt(key(INT), val(1000)))`. No stored key: `not(keyExists())`. Country is US or CA: `or(eq(stringBin("country"), val("US")), eq(stringBin("country"), val("CA")))`. Exactly one of hand=hook, leg=peg, pet=parrot: `exclusive(eq(stringBin("hand"), val("hook")), eq(stringBin("leg"), val("peg")), eq(stringBin("pet"), val("parrot")))`.

---

## Arithmetic expressions (5.6.0+)

Arithmetic and bitwise operators for numeric and integer expressions. Used in filters (e.g. comparisons on computed values) and in operation expressions (computed bins). All ops introduced in 5.6.0. See [Arithmetic](https://aerospike.com/docs/develop/expressions/arithmetic) for full API, argument types, and C/Java examples.

| Op | Purpose |
|----|--------|
| **add**, **sub**, **mul**, **div**(arg0, arg1, ...) | Addition, subtraction, multiplication, division. All number_expr; div left-to-right; single-arg div = reciprocal (or 0 for int). |
| **mod**(numerator, denominator) | Integer modulo. |
| **abs**(value), **min**, **max**(arg0, ...) | Absolute value; minimum/maximum of numeric args. |
| **floor**, **ceil**(value) | Round float down/up to nearest integer (returns float). |
| **log**(num, base), **pow**(base, exponent) | Logarithm and power; float_expr. |
| **to_int**(value), **to_float**(value) | Cast float→int / int→float. |
| **int_and**, **int_or**, **int_xor**, **int_not** | Bitwise AND, OR, XOR, NOT on integers. |
| **int_lshift**, **int_rshift**, **int_arshift**(value, by_numbits) | Left shift, logical right shift, arithmetic right shift. |
| **int_count**(value), **int_lscan**, **int_rscan**(value, search) | Bit count (popcount); scan from MSB or LSB for bit 0/1, return bit index. |

**Examples:** Filter where |x1 − x2| &gt; 1: `gt(abs(sub(intBin("x1"), intBin("x2"))), val(1))`. Sum apples + bananas &gt; 10: `gt(add(intBin("apples"), intBin("bananas")), val(10))`. Even value: `eq(mod(intBin("value"), val(2)), val(0))`. Workshop (simple-op-exp.py, roster.py) uses Sub, Add, Mod, Abs in filters and operation expressions.

---

## Record metadata expressions (5.2.0+)

Record-level metadata is stored in the primary index and is available **without reading record storage**, so metadata-based filters are fast (see [Expression execution model](#expression-execution-model)). See [Record metadata](https://aerospike.com/docs/develop/expressions/metadata) for full API and C/Java examples.

| Op | Purpose | Units / notes |
|----|--------|----------------|
| **key_exists**() | True if record has a stored key. | boolean |
| **set_name**() | Record’s set name. | string |
| **since_update**() | Time since last update. | **milliseconds** |
| **last_update**() | Last-update time from Unix epoch. | **nanoseconds** (ms resolution) |
| **ttl**() | Time to live. | **seconds** (integer) |
| **void_time**() | Expiration time. | **nanoseconds** (second resolution); −1 = never expire |
| **record_size**() | Physical record size on storage (compressed if namespace compression on). | bytes (integer). Introduced 7.0.0. |
| **digest_modulo**(mod) | Record digest mod N (e.g. for sampling). | integer |
| **is_tombstone**() | True if record is a tombstone. | boolean. XDR filters and write with read/write expressions only. |
| **device_size**(), **memory_size**() | **Deprecated** (8.1). Use **record_size** instead. | bytes; 0 for memory namespace (device_size) or non-memory (memory_size). |

**Examples:** Filter records updated &gt; 1 s ago: `gt(since_update(), val(1000))`. TTL &lt; 30 min: `lt(ttl(), val(60*30))`. Set is groupA or groupB: `or(eq(set_name(), val("groupA")), eq(set_name(), val("groupB")))`. Records &gt; 1 MiB: `gt(record_size(), val(1024*1024))`. Workshop uses SinceUpdateTime (ms) and TTL (s) in filters and as operation reads.

---

## Record storage expressions (5.2.0+)

Expressions that read **bin and key values** from the record. They require the **storage phase** when used in filters (see [Expression execution model](#expression-execution-model)). See [Record storage](https://aerospike.com/docs/develop/expressions/storage) for full API and C/Java examples.

| Op | Purpose |
|----|--------|
| **bin_int**(name), **bin_float**(name), **bin_str**(name), **bin_blob**(name) | Access bin as integer, float, string, or blob. Return **unknown** if bin missing or type mismatch. |
| **bin_list**(name), **bin_map**(name), **bin_geo**(name), **bin_hll**(name) | Access bin as list, map, GeoJSON, or HyperLogLog. Return **unknown** if bin missing or type mismatch. |
| **bin_exists**(name) | True if a bin with that name exists; otherwise **unknown**. |
| **bin_type**(name) | ParticleType of the bin (integer: NULL 0, INTEGER 1, DOUBLE 2, STRING 3, BLOB 4, JBLOB 7, BOOL 17, HLL 18, MAP 19, LIST 20, GEOJSON 23). Unknown if bin missing. |
| **key_int**(), **key_str**(), **key_blob**() | Access stored key as integer, string, or blob. Return **unknown** if key missing or type mismatch. |

**Note:** Client APIs often expose these as e.g. `Exp.intBin("name")`, `Exp.stringBin("name")`. Use **key_exists**() (metadata) to test for a stored key without reading storage; use **key_int**() etc. when you need the key value (triggers storage read in filter context).

---

## List bin operations (5.2.0+)

Expression API for **reading** and **modifying** list-type bins. Used in filters (e.g. list size, get-by-index/value) and in operation expressions (e.g. append, remove, write result to bin). All take a list bin (or nested context) and optional policy; modify ops return the **modified list bin** (unlike CDT list ops which may return a count or other type). See [List bin operations](https://aerospike.com/docs/develop/expressions/list-bin) for full API and [cdt-api.md](cdt-api.md) for CDT context (ordering, policies).

| Category | Ops | Notes |
|----------|-----|--------|
| **Modify** | list_append, list_append_items, list_clear, list_increment, list_insert, list_insert_items | **list_append** does *not* create the bin if missing (unlike the CDT list append op). |
| | list_remove_by_index, list_remove_by_index_range, list_remove_by_index_range_to_end | Remove by position. |
| | list_remove_by_rank, list_remove_by_rank_range, list_remove_by_rank_range_to_end | Remove by sorted rank. |
| | list_remove_by_rel_rank_range, list_remove_by_rel_rank_range_to_end | Remove by rank relative to a value. |
| | list_remove_by_value, list_remove_by_value_list, list_remove_by_value_range | Remove by value(s) or value range. |
| | list_set, list_sort | Set element at index; sort list. |
| **Read** | list_size | Returns integer (element count). |
| | list_get_by_index, list_get_by_index_range, list_get_by_index_range_to_end | Get by position; result_type (e.g. LIST_RETURN_VALUE) required. |
| | list_get_by_rank, list_get_by_rank_range, list_get_by_rank_range_to_end | Get by sorted rank. |
| | list_get_by_rel_rank_range, list_get_by_rel_rank_range_to_end | Get by rank relative to value. |
| | list_get_by_value, list_get_by_value_list, list_get_by_value_range | Get by value(s) or range; result_type required. |

**Note:** Read ops (except list_size) take a **result_type** (e.g. LIST_RETURN_VALUE, LIST_RETURN_INDEX); when returning values, a type is required because list elements can be any expression type. Workshop examples (messages.py, roster.py, exp-composition.py) use ListSize, ListGetByIndex, ListGetByValue, ListGetByValueRange, ListSort, ListAppendItems, and related ops in filters and operation expressions.

---

## Map bin operations (5.2.0+)

Expression API for **reading** and **modifying** map-type bins. Used in filters (e.g. map size, get-by-key/value) and in operation expressions (e.g. put, remove, write result to bin). All take a map bin (or nested context) and optional policy; modify ops return the **modified map bin** (unlike CDT Map ops which may return a count or other type). See [Map bin operations](https://aerospike.com/docs/develop/expressions/map-bin) for full API and [cdt-api.md](cdt-api.md) for CDT context (ordering, policies).

- **map_put** and **map_put_items** do *not* create a new bin if the specified bin does not exist (unlike the CDT Map put/put_items ops, which do create the bin).
- Map **read** expressions (other than **map_size**) return a result based on their **result_type** (integer_value). Single-result ops (e.g. map_get_by_index, map_get_by_key) require a type when result_type is MAP_RETURN_KEY or MAP_RETURN_VALUE, because map keys/values can be any expression type. **map_size** returns an integer (element count).

| Category | Ops | Notes |
|----------|-----|--------|
| **Modify** | map_clear, map_put, map_put_items, map_increment | Put/put_items do not create bin if missing. |
| | map_remove_by_index, map_remove_by_index_range, map_remove_by_index_range_to_end | Remove by position. |
| | map_remove_by_key, map_remove_by_key_list, map_remove_by_key_range | Remove by key(s) or key range (start ≤ k &lt; end). |
| | map_remove_by_rank, map_remove_by_rank_range, map_remove_by_rank_range_to_end | Remove by sorted rank. |
| | map_remove_by_rel_index_range, map_remove_by_rel_index_range_to_end | Remove by index relative to key. |
| | map_remove_by_rel_rank_range, map_remove_by_rel_rank_range_to_end | Remove by rank relative to value. |
| | map_remove_by_value, map_remove_by_value_list, map_remove_by_value_range | Remove by value(s) or value range. |
| **Read** | map_size | Returns integer (entry count). |
| | map_get_by_index, map_get_by_index_range, map_get_by_index_range_to_end | Get by position; result_type required. |
| | map_get_by_key, map_get_by_key_list, map_get_by_key_range | Get by key(s) or key range. |
| | map_get_by_rank, map_get_by_rank_range, map_get_by_rank_range_to_end | Get by sorted rank. |
| | map_get_by_rel_index_range, map_get_by_rel_index_range_to_end | Get by index relative to key. |
| | map_get_by_rel_rank_range, map_get_by_rel_rank_range_to_end | Get by rank relative to value. |
| | map_get_by_value, map_get_by_value_list, map_get_by_value_range | Get by value(s) or range; result_type required. |

---

## Blob/bitwise bin operations (5.2.0+)

Expression API for **reading** and **modifying** blob-type bins at bit or byte level. Used in filters and operation expressions. The [Blob bin operations](https://aerospike.com/docs/develop/expressions/blob-bin) page documents **bit_*** ops that operate on a region of the blob (bit_offset/bit_size or bytes_offset/byte_size). Modify ops return the **modified blob bin**; read ops return blob or integer as noted. See the official [Blob data type](https://aerospike.com/docs/develop/data-types/blob) for underlying add/set/get/count/scan semantics.

| Category | Ops | Notes |
|----------|-----|--------|
| **Modify** | bit_add, bit_subtract | Add/subtract integer in a bit region; action_flags. |
| | bit_and, bit_or, bit_xor, bit_not | Bitwise AND, OR, XOR, NOT on a bit region. |
| | bit_set, bit_set_int | Set bit region from blob or integer. |
| | bit_lshift, bit_rshift | Left/right shift a bit region. |
| | bit_insert, bit_remove | Insert/remove bytes at byte offset. |
| | bit_resize | Resize blob (bytes_size, flags). |
| **Read** | bit_count | Popcount in region → integer_bin. |
| | bit_get, bit_get_int | Get bit region as blob or integer (is_signed for get_int). |
| | bit_lscan, bit_rscan | Scan for first 0/1 from left or right → integer_bin (bit index). |

---

## HyperLogLog bin operations (5.2.0+)

Expression API for **reading** and **modifying** HyperLogLog (HLL) bins used for approximate distinct counting and similarity. Used in filters (e.g. **hll_get_count** in a cond or comparison) and in operation expressions. See [HyperLogLog bin operations](https://aerospike.com/docs/develop/expressions/hll-bin) for full API and the [HyperLogLog data type](https://aerospike.com/docs/develop/data-types/hll) for semantics.

| Category | Ops | Notes |
|----------|-----|--------|
| **Modify** | hll_add | Add values (list_expr) to HLL; index_bit_count; returns hll_bin. |
| | hll_add_mh | Add with MinHash; index_bit_count, minhash_bit_count. |
| | hll_update | Update HLL with values (list_expr). |
| **Read** | hll_get_count | Estimated distinct count → integer_bin. |
| | hll_get_union_count | Union count of HLLs (hll_list, bin) → integer_bin. |
| | hll_get_intersect_count | Intersect count (hll_list, bin) → integer_bin. |
| | hll_get_similarity | Similarity (hll_list, bin) → float_bin. |
| | hll_get_union | Union of HLLs → hll_bin. |
| | hll_describe | Describe HLL (index/minhash bits etc.) → list_bin. |
| | hll_may_contain | 1 if bin may contain all values, else 0 → integer_bin. |

**Example (filter):** Workshop “let” example uses **hll_get_count** in a filter: e.g. distinct count &lt; 1k or &gt; 100M. Use **hll_may_contain** in filters to test set containment without full scan.

---

## Nested CDT expressions

You can **nest** list and map expression ops to reach elements inside lists and maps (e.g. list → map → list). The output of an inner get (e.g. `ListExp.getByIndex`, `MapExp.getByKey`) is used as the bin/context for an outer get or is passed into a comparison (e.g. `regexCompare`).

**Pattern:** Chain from the bin inward — e.g. `ListExp.getByIndex(..., 3, Exp.listBin("bin4"))` yields the fourth list element (a map); pass that into `MapExp.getByKey(..., 2, ...)` to get the value at map key `2`; wrap in `Exp.regexCompare("str.*3", flags, ...)` to use in a filter. Deeper nesting: map key `"list1"` → list → `ListExp.getByIndex(..., 0, ...)` for the first list element, then compare.

**Use case:** Filter or read by a value buried in a nested structure without path expressions (8.1.1+). Same idea as CDT context in the client API (fixed path); expressions compose the path from list/map get ops.

**Reference:** [Nested collection data type expressions](https://aerospike.com/docs/develop/expressions/nesting) (official docs; comprehensive tutorial covering CDT operations, expression composition, and path expressions together).

### Path expressions inside the expression API (8.1.1+)

Path expressions can be used inside the expression API (`CdtExp.selectByPath`, `CdtExp.modifyByPath`), not just as direct CDT operations (`CdtOperation.selectByPath`). This is a significant distinction:

- **`CdtExp.selectByPath`** evaluates inside an expression context. It can be used as part of a filter expression (where the outermost expression must be boolean) or as the indexed value in a secondary index expression definition. The result is a typed value (e.g. list) that can feed into further expression composition (e.g. `ListExp.getByValue` with `EXISTS` to test membership across all elements).
- **`CdtExp.modifyByPath`** modifies nested elements within an expression context and returns the modified collection. Used in operation expressions.
- **`CdtOperation.selectByPath` / `CdtOperation.modifyByPath`** are CDT operations used directly in `operate()` calls, including query projection. They act on the record in-place and return data under a bin name.

**Constraint:** `AND_FILTER` context (8.1.2) is not supported inside `CdtExp.selectByPath`. Use `CdtOperation.selectByPath` instead when `AND_FILTER` is needed.

---

## 8.1.2 additions

Aerospike Database 8.1.2 adds three standalone expression operators (AER-6889) that simplify common patterns previously requiring multi-op composition. Each is a top-level `Exp` function, not a nested CDT op, and does not take context or return-type arguments.

| Operator | Signature | Returns | Replaces |
|----------|-----------|---------|----------|
| `Exp.inList(value, list)` | value-expr, list-expr | boolean | `ListExp.getByValue(COUNT, value, list) > 0` |
| `Exp.mapKeys(bin)` | map-bin-expr | list of keys | `MapExp.getByKeyRange(KEY, nil, nil, bin)` or equivalent |
| `Exp.mapValues(bin)` | map-bin-expr | list of values | `MapExp.getByValueRange(VALUE, nil, nil, bin)` or equivalent |

**When to reach for each:**

- **`in_list`** — boolean membership test. The canonical use is a server-side "does this list contain X?" filter: blocklist checks, liker/follower membership, duplicate-action detection. Replaces the COUNT-greater-than-zero idiom with a single boolean op, which is both clearer to read and expressible as a one-line filter expression attached to a read, write, or operate.
- **`map_keys`** — extract the full key list of a map in a filter or projection without needing to supply a context/return-type tuple. Composes naturally with `list_size`, `list_get_by_index`, and `in_list` (e.g., "does this map have any of these keys?").
- **`map_values`** — same shape as `map_keys` for values. Useful for predicates like "is any value in the map greater than X?" in one compact expression.

These operators are **filter-expression primitives** — they do not require path expressions. They work on every DB 8.1.2+ cluster with a client library that exposes them, and compose with the existing `Exp` / `ListExp` / `MapExp` / `CdtExp` vocabulary.

**Related 8.1.2 path-expression ergonomics** (`CTX.mapKeysIn`, `CTX.andFilter`) are covered in [path-expressions.md](path-expressions.md).

**Operation projection in queries (AER-6892)** — foreground secondary-index queries now accept the same bin projection, read operations, CDT operations, and read expressions used in single-key `operate` calls. Set these on the query via the client's statement / operations API (e.g. `Statement.setOperations(...)` in Java, `query.add_ops` or equivalent in Python). Key properties:

- **Read-only** on foreground queries. Background writes via queries still use `query_apply` / ops-bg with write ops or a UDF.
- Replaces the **two-phase "query then batch-get"** idiom: the server returns the projected/computed bins per matching record directly. Most valuable when the matched record is large but the projection is small.
- Backward compatible: projecting a list of bin names continues to work.
- Path expressions may be used in the projection, not only in the filter expression — useful for returning just the subtree of a nested document that matches a path.

The feature does not help key-based reads or batch-of-known-keys paths, which already support operations.

---

## Expression execution model

- **Metadata** lives in the **primary index** (64-byte entry per record). **Record data** lives on the namespace storage device (SSD, PMem, RAM). Resolving expressions using only metadata avoids disk I/O and can yield an **order of magnitude** performance improvement.
- **Two-phase evaluation (filter expressions):**
  1. **Metadata phase (fast path):** Resolve using only primary-index metadata. Any sub-expression that reads storage data evaluates to **unknown**. **Trilean logic:** logical expressions can sometimes return definite true/false even with unknown inputs; most other expressions propagate unknown. Outcomes: **false** → record filtered out, no storage read; **true** → operation proceeds (storage read only if needed); **unknown** → phase 2.
  2. **Storage phase:** Load the full record (physical I/O if on disk), re-evaluate the expression. Result is always definite true or false.
- **Implication:** Put **metadata conditions** (e.g. TTL, since_update_time) in the filter when possible so that failing records are discarded in the metadata phase and storage is not read. Especially important when data is on SSD/PMem/flash.
- The two-phase model above applies to record filter evaluation (queries, scans, batch reads, filter policies). Operation expressions and XDR filter expressions follow the same trilean rules for how `unknown` propagates; see each feature's documentation for how results are applied.

---

## Expressions vs UDFs

- **Expressions** are the preferred way to do server-side filtering and computed bin operations. They perform better and scale better than Lua UDFs. The expression API has full access to the server’s list/map (and other) APIs; the Lua UDF API is older and does not expose the full CDT API (e.g. no HyperLogLog, bitwise ops, or full composition). Replacing UDFs with filter and operation expressions is recommended where possible.
- **Path expressions** (8.1.1+; 8.1.2 adds `mapKeysIn` and `andFilter` contexts) add **iteration** over list/map elements and expression-based selection (see [path-expressions.md](path-expressions.md)). The Summit talk noted that **iterating over map keys and performing the same operation on each** is not supported in (classic) expressions and was under consideration but likely would not be added there — path expressions address that use case with selectByPath/modifyByPath and loop variables.

---

## Debugging

- The server returns **error codes**, not human-readable messages. Generic messages are filled in by the client. Detailed information is in the **server log** (e.g. warning level). For development, use a lower log level (debug/detail) and/or configure Aerospike to send expression (and partition) context to a separate log file with its own log level. See Aerospike documentation under configuration and log files. Operation expressions can result in error code 26 (not applicable) when a cond or other expression evaluates to unknown and the policy does not suppress it.

---

## Relation to other research

- **Data modeling ([concepts-and-patterns.md](concepts-and-patterns.md)):** Filter expressions are the WHERE clause for scans and queries; they affect which records are read, updated, or returned in batch. Operation expressions support “record as document” patterns (computed bins, cross-bin updates) without UDFs. Set indexes (5.6) improve “all records in set S” in large namespaces.
- **CDT API ([cdt-api.md](cdt-api.md)):** The expression language exposes the same list/map (and other) operations; operation expressions compose them and add arithmetic, comparison, and control flow (let, cond).
- **Path expressions ([path-expressions.md](path-expressions.md)):** Path expressions (8.1.1+) provide iteration over nested CDTs and expression-based filtering with loop variables; they complement (classic) filter and operation expressions for document-style models.

---

## Takeaways for a canonical example application

- Use **filter expressions** for scan and query WHERE clauses (and batch read filtering). Favor metadata (TTL, since_update_time) in filters when possible to avoid storage reads.
- Use **operation expressions** for computed bins and atomic cross-bin updates instead of UDFs when the logic fits (let, cond, composition of list/map ops). Enables “record as document” updates in a single operate transaction.
- **Set index** (if available for the server version in use) can speed up “all records in set S” when the set is a small fraction of the namespace.
- Use filter expressions on writes to **control** whether an operation proceeds (conditional write).
- **Secondary index expressions** (8.1+): when a secondary index on a computed value is more memory-efficient than indexing a raw bin, create an expression index via asadm and query by that expression or index name. Use `cond(condition, value_to_index, unknown())` to exclude records from the index (sparse index); see [Secondary index expressions (expression indexes)](#secondary-index-expressions-expression-indexes) and the [Expression index tutorial](https://aerospike.com/docs/database/learn/tutorials/quick-start-expression-index/).
- When debugging expression failures, check server logs and consider expression-specific log context and log level.
