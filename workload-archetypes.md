# Aerospike Workload Archetypes

**Summary:** Representative customer schemas and workloads described in terms of record structure and write operations. Each archetype documents what the record looks like (key, bins, types, sizes) and how it is modified (which bins change, what operations are used, how much data moves per write).

**Premise:** In a highly customizable database like Aerospike, a single "average" workload is a myth — an ad-tech profile store looks completely different from a financial fraud detection system or a gaming session store. Categorizing workloads into distinct archetypes based on data models and usage patterns is a more accurate and useful approach for benchmarking.

---

## Summary Matrix

| Archetype                                          | Record Size                    | Bins                          | CDT                           | Write Operation                              | Data Modified per Write             | Write Amplification                                                 |
| -------------------------------------------------- | ------------------------------ | ----------------------------- | ----------------------------- | -------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------- |
| **A1** Blob Store                                  | Few KiB – tens of KiB          | 1                             | None                          | `put(REPLACE)` — full record                 | All (entire blob)                   | 1:1 (payload = record)                                              |
| **A2** Counters                                    | Sub-100 B                      | 1                             | None                          | `add()` — atomic increment                   | 8 bytes                             | Low (record is tiny)                                                |
| **B** Multi-Bin Scalar                             | 100 B – few KiB                | 5–20                          | None                          | `operate()` — 1–3 bin puts                   | Tens of bytes                       | Low-moderate (record is small)                                      |
| **B†** Index-dominated micro-entity (variant of B) | ~100–700 B (cluster-dependent) | Few–30 scalars, or 1 map + ID | Optional map bin (Trust)      | `put()` / `delete()` + partial bin puts      | Tens of bytes                       | **High PI/CPU** at billions of objects; payload rewrite still small |
| **B2** Extreme Multi-Bin                           | ~50 KiB                        | Hundreds–thousands            | None                          | `operate()` — 1–few bin puts                 | Tens of bytes                       | High (50 KiB rewrite) + O(n²) bin merge CPU. Fixed by 8.1.1         |
| **C** Document/CDT                                 | 1–10 KiB                       | 1–5                           | Map/List/Nested               | CDT ops or full bin replace                  | Element (tens of bytes) or full CDT | Moderate                                                            |
| **G** Segment Map                                  | Few KiB – tens of KiB          | 1                             | K-ordered map                 | `map_put` (upsert)                           | ~20 bytes                           | Moderate-high                                                       |
| **H** Leaderboard                                  | Tunable per record             | 1                             | K-ordered map (composite key) | `map_remove` + `map_put`                     | ~60 bytes (remove + insert)         | High (large map rewritten)                                          |
| **E** Association Lists                            | 1–150 KiB                      | 1                             | Ordered list w/ persistIndex  | `list_append(ADD_UNIQUE)`                    | ~15 bytes                           | High at scale (150 KiB rewrite for 15 B)                            |
| **F** Time-Series Roll-Up                          | 0 → ~10 KiB (grows)            | 1                             | List of tuples                | `list_append`                                | ~7 bytes                            | Grows through time slice                                            |
| **D** Consolidated Hierarchy                       | 50–350 KiB                     | 4–6                           | Nested maps + aux lists/maps  | Multi-CDT `operate()`                        | ~400 bytes new data                 | High (250 KiB rewrite for 400 B)                                    |
| **I** Event Timeline                               | 0 → ~40 KiB (grows)            | 1                             | List of maps                  | `list_append` / `modify_by_path`             | ~160 bytes (append) or bulk modify  | Grows through day                                                   |
| **J** Multi-Bin Entity                             | 5–30 KiB                       | 8–12                          | 2–4 ordered lists + scalars   | `operate()` — mixed (increment, append, put) | 8–36 bytes per event                | High (26 KiB rewrite for 8 B)                                       |

---

## Archetype A1: Single-Bin Blob Store (Client-Compressed / Serialized Object)

A single bin containing a serialized object or application-level compressed blob, accessed as an opaque unit.

### Representative Use Cases

- Ad-tech user profile (Protobuf/Avro-serialized segment map)
- Session state store (Java/Python serialized session object)
- ML feature vectors (compressed embedding arrays)
- Document cache (pre-rendered HTML or JSON blobs)

### Record Structure

| Attribute       | Value                                           |
| --------------- | ----------------------------------------------- |
| **Key**         | Entity ID (user ID, session token, document ID) |
| **Record size** | Medium to large (few KiB to tens of KiB+)       |
| **Bin count**   | 1 (occasionally 2 with a version/metadata bin)  |
| **Bin types**   | Bytes or String — opaque to the server          |
| **CDT usage**   | None                                            |

### Write Operations

| Attribute                            | Value                                                                                                                        |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `put()` with `REPLACE` policy — complete record replacement                                                                  |
| **Bytes written per write**          | Entire record (few KiB to tens of KiB)                                                                                       |
| **Bins modified per write**          | All (the single bin is fully overwritten)                                                                                    |
| **Concurrency control**              | Check-And-Set (CAS) via generation count — client reads generation, writes with expected generation; server rejects if stale |
| **Write frequency**                  | Varies — often balanced or write-heavy relative to reads                                                                     |
| **Server-side computation on write** | None — server stores the blob as-is                                                                                          |

### Read Operations

| Attribute          | Value                                                |
| ------------------ | ---------------------------------------------------- |
| **Read operation** | `get()` — full record                                |
| **Bytes returned** | Entire record                                        |
| **Read frequency** | Proportional to writes (balanced) or somewhat higher |

### What Makes This Archetype Distinct

- The server never inspects the bin contents. All serialization/deserialization happens in the application.
- Every write replaces the **entire** record — there are no partial updates. A 1-byte change in the serialized object still rewrites the full payload.
- Generation-based CAS is the only concurrency mechanism; without it, last-writer-wins applies.
- Record size depends entirely on the application's encoding — the same logical data can be 2 KiB (compressed Protobuf) or 50 KiB (verbose JSON).

---

## Archetype A2: High-Frequency Single-Bin Counters (Minimal Scalar Store)

A single bin storing a numeric type, updated at extremely high frequency via server-side arithmetic.

### Representative Use Cases

- Rate limiters (requests per second per API key)
- View/impression counters (page views, ad impressions)
- Distributed semaphores and quota trackers
- Real-time voting/polling systems

### Record Structure

| Attribute       | Value                                                                                                          |
| --------------- | -------------------------------------------------------------------------------------------------------------- |
| **Key**         | Composite: entity + time window (e.g., `apikey123:20260604T1700`)                                              |
| **Record size** | Extremely small (sub-100 bytes on storage including metadata overhead)                                         |
| **Bin count**   | 1 (or 2 — counter + a metadata flag)                                                                           |
| **Bin types**   | Integer (8 bytes)                                                                                              |
| **CDT usage**   | None                                                                                                           |
| **TTL**         | Often set — record auto-expires when the time window closes (e.g., 60-second TTL for per-minute rate limiting) |

### Write Operations

| Attribute                            | Value                                                                      |
| ------------------------------------ | -------------------------------------------------------------------------- |
| **Write operation**                  | `add()` — server-side atomic increment (or decrement)                      |
| **Bytes written per write**          | 8 bytes (the integer bin) — though the full record is rewritten on storage |
| **Bins modified per write**          | 1                                                                          |
| **Concurrency control**              | None needed — `add()` is inherently atomic under the record lock           |
| **Write frequency**                  | Extremely high — hundreds to thousands of writes/second to a single key    |
| **Server-side computation on write** | Arithmetic only (read current value, add delta, store)                     |

### Read Operations

| Attribute          | Value                                                                            |
| ------------------ | -------------------------------------------------------------------------------- |
| **Read operation** | `get()` for current value, or read within an `operate()` alongside the increment |
| **Bytes returned** | 8 bytes                                                                          |
| **Read frequency** | Much lower than writes (10:1 to 100:1 write:read ratio)                          |

### What Makes This Archetype Distinct

- Extremely hot keys — a single record may sustain thousands of writes/second.
- Primary index cost dominates: 64 bytes of PI metadata for an 8-byte payload.
- When a key becomes too hot (KEY_BUSY errors), the shard-on-demand pattern distributes across sub-records: each sub-record holds a partial counter, reads sum all sub-records.
- Every write touches the minimum possible data (one integer), but the server still performs a full copy-on-write of the record internally.
- Record TTL is the primary lifecycle mechanism — counters expire automatically when the time window passes.

---

## Archetype B: Multi-Bin Scalar Store (Partial Updates / Merges)

Multiple bins containing flat scalar types, updated independently via partial writes that merge on the server.

### Representative Use Cases

- User preference/settings records (timezone, language, theme, notification prefs)
- Device state records (last_seen, firmware_version, battery_level, location)
- Order/transaction status records (status, amount, timestamp, handler_id)
- Configuration records (feature flags, thresholds, toggle states)
- Telco charging/session records (multi-bin binding and profile sets, ~600 B)
- Identity/trust platform — **B†** index-dominated micro-entities (Trust, Key, Search clusters; billions of small records)

### Record Structure

| Attribute       | Value                                        |
| --------------- | -------------------------------------------- |
| **Key**         | Entity ID (user ID, device ID, order ID)     |
| **Record size** | Small (few hundred bytes to few KiB)         |
| **Bin count**   | 5–20                                         |
| **Bin types**   | All scalar: integers, doubles, short strings |
| **CDT usage**   | None                                         |

### Write Operations

| Attribute                            | Value                                                                                                                           |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `operate()` with 1–3 bin puts per call; or single-bin `put()` for one field                                                     |
| **Bytes written per write**          | Only the modified bins (tens of bytes), but the full record is rewritten on storage                                             |
| **Bins modified per write**          | 1–3 out of 5–20 total                                                                                                           |
| **Concurrency control**              | Usually none — different writers update different bins, or single-owner per entity. Occasional CAS for multi-writer contention. |
| **Write frequency**                  | Moderate — state changes drive writes (e.g., device heartbeat every 30s updates `last_seen`)                                    |
| **Server-side computation on write** | Bin merge — server reads existing record, applies only the specified bin operations, writes the combined result                 |

### Read Operations

| Attribute          | Value                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------- |
| **Read operation** | Full `get()` or bin projection (`get()` requesting specific bin names)                      |
| **Bytes returned** | Full record (few hundred bytes) or projected subset                                         |
| **Read frequency** | Higher than writes for display-oriented records; can be balanced for state-tracking records |

### Customer Example: Telco charging / session data

An anonymized telco session-data deployment. Runs in AP mode at high TPS. Multi-bin scalar records with partial bin updates and write/delete-heavy traffic.

#### Record Structure

| Attribute        | Value                                                            |
| ---------------- | ---------------------------------------------------------------- |
| **Key**          | Product-specific entity/session identifiers                      |
| **Record size**  | **~617 bytes** average; many records in **128–512 B** range      |
| **Bin count**    | Multiple scalar bins per record (typical B range; varies by set) |
| **Bin types**    | Integers, short strings, timestamps                              |
| **CDT usage**    | None on the dominant path                                        |
| **Object count** | **200M** records                                                 |

#### Write Operations

| Attribute                            | Value                                                                                                 |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `put()` / `operate()` on individual bins; high-volume `delete()` in steady-state mix                  |
| **Bytes written per write**          | Tens of bytes per changed bin; full ~600 B record rewritten on storage                                |
| **Bins modified per write**          | Typically 1–few bins per event                                                                        |
| **Concurrency control**              | Not documented — independent fields per service                                                       |
| **Write frequency**                  | **Write/delete-heavy** — lab: **~49.7k TPS** write/delete vs **~20k** SI-query TPS (70k total target) |
| **Server-side computation on write** | Bin merge only                                                                                        |

#### Read Operations

| Attribute          | Value                                                                                                               |
| ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| **Read operation** | Short secondary-index queries with filter expressions                                                               |
| **Bytes returned** | Full record (~600 B) per query hit                                                                                  |
| **Read frequency** | **~29%** of TPS (SI queries); disproportionate CPU cost (~5 vCPU for ~9k SI TPS vs ~1.5 vCPU for ~50k write/delete) |

### Customer Example: Identity/trust platform — B† index-dominated micro-entity

An anonymized identity/trust platform runs multiple All-Flash clusters holding **identity and trust** data at **tens of billions of objects** per major cluster. Records are structurally Archetype B (multi-field entities with partial updates), but operationally behave as **B†**: the **primary index and CPU cost per object** dominate benchmarking and sizing more than bytes moved per write.

#### What distinguishes B† from generic B

| Dimension          | Generic B                   | B†                                                                 |
| ------------------ | --------------------------- | ------------------------------------------------------------------ |
| **Sizing driver**  | Record bytes + bin merge    | **Objects/node × flash PI × write bursts**                         |
| **Record size**    | Few hundred B – few KiB     | Often **sub-200 B** (Trust ~119 B)                                 |
| **Scale**          | Millions–low billions       | **45–60B+** per mega-cluster                                       |
| **Lifecycle**      | Often TTL or moderate churn | **No TTL** on main datasets; durable delete                        |
| **Failure sizing** | Moderate headroom           | Plan for **~2× load** under rack loss + rebuild (RF=2, rack-aware) |

#### Record structure by cluster

| Cluster                 | Avg record size                    | Object count                       | Bin / CDT model                | TTL               |
| ----------------------- | ---------------------------------- | ---------------------------------- | ------------------------------ | ----------------- |
| **Trust Features**      | **~119 B** (~79% in 64–128 B band) | **~58B** today → **100B+** planned | Few scalars and/or one map bin | **None**          |
| **Key Data**            | **~519 B**                         | **45B** baseline, growing          | Multi-bin scalar (typical B)   | **None**          |
| **Search Data**         | **~659 B**                         | **~60B**                           | Multi-bin scalar               | Not detailed      |
| **Device Intelligence** | **~512 B** weighted avg            | **~3.8B** total                    | Per-set sizes below            | Used in PI sizing |

**Device Intelligence sets:**

| Set                      | Records | Avg size     |
| ------------------------ | ------- | ------------ |
| `lsh_bucket` (amortized) | 2.5B    | 138 B        |
| `token_map`              | 500M    | 230 B        |
| `local_device_id_map`    | 500M    | 190 B        |
| `device_alias`           | 200M    | 152 B        |
| `device_entity`          | 100M    | **13.3 KiB** |

#### Write operations (Trust Features — anchor)

| Attribute                   | Value                                                                                                       |
| --------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Write operation**         | `put()` / partial bin updates; durable `delete()`                                                           |
| **Bytes written per write** | Tens of bytes logical; full **~119 B** record rewritten on storage                                          |
| **Bins modified per write** | Typically 1–few                                                                                             |
| **Write frequency**         | **Burst-heavy** — P50 **293** wps; P95 **~45.5k** wps; P99 **~61.9k**; peak **~92.6k**; target **125k** wps |
| **Server-side computation** | Bin merge only                                                                                              |

#### Read operations (Trust Features — anchor)

| Attribute          | Value                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Read operation** | Full `get()` / batch reads                                                                                       |
| **Bytes returned** | Full micro-record (~119 B)                                                                                       |
| **Read frequency** | P50 **571** rps; P95 **~6.4k** rps; peak **~10.8k** rps — **much lower than write bursts** (~7:1 P95 write:read) |

### What Makes This Archetype Distinct

- Writes touch a small subset of bins — the application doesn't need to know the full record contents to update one field.
- Multiple independent writers can update different bins on the same record without conflict (e.g., one service updates `battery_level`, another updates `firmware_version`).
- The record on storage is always a complete copy — "partial write" means the server merges the new bins into the existing record, not that only a fragment is stored.
- Bin projection on reads keeps network payload small when only a few fields are needed.
- **B† variant (identity/trust platform):** At billions of **sub-200 B** records, **primary-index footprint and CPU per lookup** dominate — not payload size. Benchmarks must include **object count, flash PI, write bursts, durable deletes/tombstones, and degraded-mode rack failure**, not average bytes/record alone.

---

## Archetype B2: Extreme Multi-Bin Scalar Store (Thousands of Bins)

A variant of Archetype B where the application uses hundreds to thousands of independent scalar bins per record, treating each bin as a named column. The record is large (~50 KiB) not because any single bin is large, but because there are thousands of them.

### Representative Use Cases

- Ad-tech user attribute stores (one bin per attribute/signal, accumulated over time)
- Feature stores for ML (one bin per feature, updated independently by different pipelines)
- Wide-column migrations from Cassandra/HBase (direct column-to-bin mapping)
- Telemetry aggregation (one bin per metric dimension)

### Record Structure

| Attribute       | Value                                                                   |
| --------------- | ----------------------------------------------------------------------- |
| **Key**         | Entity ID (user ID, device ID)                                          |
| **Record size** | ~50 KiB (driven by thousands of small bins, not any single large value) |
| **Bin count**   | Hundreds to thousands (observed: up to 9,444 bins per record)           |
| **Bin types**   | All scalar: integers, short strings, doubles                            |
| **CDT usage**   | None — every value is a separate named bin                              |

### Write Operations

| Attribute                            | Value                                                                                                                                                                                   |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `operate()` or `put()` updating 1–few bins per call                                                                                                                                     |
| **Bytes written per write**          | Tens of bytes of new/changed data per write. Full ~50 KiB record rewritten on storage.                                                                                                  |
| **Bins modified per write**          | 1–few out of thousands                                                                                                                                                                  |
| **Concurrency control**              | Typically none — different writers update different bins                                                                                                                                |
| **Write frequency**                  | Moderate under normal operation                                                                                                                                                         |
| **Server-side computation on write** | Bin merge — server must iterate the existing record's bin list to find and merge the updated bin(s). Cost is **O(n²)** where n = number of bins due to per-bin comparison during merge. |

**Write latency impact (measured):**

| Bin Count | Write-master Latency (no bin convergence) | Write-master Latency (with XDR bin convergence) |
| --------- | ----------------------------------------- | ----------------------------------------------- |
| 10–20     | Sub-millisecond                           | Sub-millisecond                                 |
| ~1,000    | Tens of milliseconds                      | Tens to hundreds of milliseconds                |
| ~10,000   | 500ms–1,024ms                             | 1s–2s                                           |

**Bin convergence:** During XDR replication, the destination server must compare each incoming bin against the existing record's bins to determine which to keep (last-writer-wins per bin). With thousands of bins, this comparison dominates write latency — 4–8 seconds observed in production at 9,444 bins.

### Read Operations

| Attribute          | Value                                                                                |
| ------------------ | ------------------------------------------------------------------------------------ |
| **Read operation** | Full `get()` (returns all thousands of bins) or bin projection (specific bin subset) |
| **Bytes returned** | Full record: ~50 KiB. Projected: tens of bytes per requested bin.                    |
| **Read frequency** | Varies — profile lookups, feature retrieval                                          |

### What Makes This Archetype Distinct

- The pathology is NOT record size — 50 KiB is well within normal operating range. The pathology is the **number of bins** and the O(n²) cost of per-bin merge/convergence on writes.
- Write latency is driven entirely by CPU (bin-by-bin iteration and comparison), not by storage I/O. Standard diagnostics (device latency, throughput, fabric) show no anomaly — only write-master latency spikes.
- XDR replication (especially rewind/catch-up) amplifies the problem: bin convergence doubles the already-high merge cost.
- This pattern is structurally identical to Archetype B (partial scalar updates), but crosses a performance cliff somewhere above ~100 bins where the bin-merge cost starts to dominate.
- The recommended alternative is consolidation into a single K-ordered Map bin (Archetype C or G) — `map_put` for updates, `map_get_by_key` for reads — which replaces the O(n²) bin merge with O(log n) map operations.

### Source

Production incident at a large-scale web platform (internal support case). Cluster: 100+ nodes, migrating via XDR. Records with 9,444 bins caused write-master latencies of 4–8 seconds on the destination cluster during XDR rewind, with no storage I/O anomaly.

---

## Archetype C: Document / CDT Store (and Mixed Patterns)

Records containing Collection Data Types (Maps, Lists) — from simple single CDTs to deeply nested structures — often combined with scalar metadata bins.

### Representative Use Cases

- Shopping carts (list of item maps with quantity, price, options)
- User vehicles/addresses/payment methods (list of maps per user)
- Product catalogs (nested map of attributes, variants, pricing tiers)
- Configuration templates (hierarchical settings as nested maps)

### Record Structure

| Attribute         | Value                                                                                                    |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| **Key**           | Entity ID (user ID, product ID, cart ID)                                                                 |
| **Record size**   | 1–10 KiB typical                                                                                         |
| **Bin count**     | 1–5 (CDT bins + optional scalar metadata)                                                                |
| **Bin types**     | Map, List, or nested combinations; plus optional integers/strings                                        |
| **CDT usage**     | Core of the archetype — Maps (K-ordered), Lists (ordered/unordered), nested (Map of Lists, List of Maps) |
| **Nesting depth** | 1–3 levels                                                                                               |

### Write Operations — Two Distinct Styles

**Style (a): CDT API Manipulation (surgical server-side edits)**

| Attribute                            | Value                                                                                                                                   |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `operate()` with CDT operations: `map_put`, `map_remove_by_key`, `list_append`, `list_remove_by_value`, nested context ops              |
| **Bytes written per write**          | Small — one element added/removed/modified within the CDT (tens to hundreds of bytes), though the entire record is rewritten on storage |
| **Bins modified per write**          | 1 (the CDT bin being manipulated)                                                                                                       |
| **Concurrency control**              | Atomic within the record — CDT operations execute under the record lock                                                                 |
| **Server-side computation on write** | CDT structure traversal + element insertion/removal/update                                                                              |

**Style (b): Full CDT Overwrite (client builds, server stores)**

| Attribute                            | Value                                                         |
| ------------------------------------ | ------------------------------------------------------------- |
| **Write operation**                  | `put()` replacing the entire Map or List bin                  |
| **Bytes written per write**          | The entire CDT (hundreds of bytes to several KiB)             |
| **Bins modified per write**          | 1 (the CDT bin, fully replaced)                               |
| **Concurrency control**              | CAS via generation (same as Archetype A1) or last-writer-wins |
| **Server-side computation on write** | None — stores the blob as-is                                  |

### Read Operations

| Attribute          | Value                                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **Read operation** | Full `get()`, or `operate()` with CDT sub-operations (`map_get_by_key`, `list_get_by_index_range`, path expressions on 8.1.2+) |
| **Bytes returned** | Full record or projected CDT subset                                                                                            |

### What Makes This Archetype Distinct

- Customers split into two camps: those using CDT API operations for surgical modifications (the server does the work), and those who treat the CDT as an opaque structure they overwrite entirely (the client does the work).
- For Style (a), the network payload per write is tiny (one element), but the server internally rewrites the full record.
- Path expressions (Aerospike 8.1.2+) enable filtering inside nested CDTs without reading the full record to the client.
- Record growth from unbounded list/map appends is the main operational risk — must have overflow strategy.

### Customer Example: Web portal session store — `browsers-primary` / `users` map (session & browser state)

An anonymized web portal runs a multi-set session and identity store on Aerospike Community **4.3.1.4** (6 nodes, RF=2, device storage). The **dominant data path** for archetype purposes is set **`browsers-primary`**: roughly **58M** master objects (**64%** of the namespace) holding browser/session associations in a **`users` Map bin**, with **MAPKEYS** secondary indexes for lookup by numeric user id. The same `users` + MAPKEYS pattern appears on smaller `browsers-*` sets (ad-platform API and storage variants, admin variants, etc.); **`user-primary`** and **`rtid-primary`** are separate high-volume sets in the same namespace and are not documented here.

**Sources:** production cluster snapshot (2025-02-28) and internal CST engagement records (TAM introduction; 2025-06-11 and 2025-09-04 weeklies). Set and namespace names are anonymized.

#### Record structure (`browsers-primary`)

| Attribute             | Value                                                                                                                                                                                                             |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Key**               | Browser / session entity id (per application naming in set prefix `browsers-primary`)                                                                                                                             |
| **Set**               | `browsers-primary`                                                                                                                                                                                                |
| **Record size**       | **p50 ~383 B**, **p90 ~639 B** (namespace objsz histogram, 2025-02-28); **~67%** of records in **256–512 B** band. Namespace avg **~289 B** unique/master (26.031 GiB / 90.165M masters) — dominated by this set. |
| **Bin count**         | Low per record; namespace registers **308** distinct bin names across all sets                                                                                                                                    |
| **Bin types**         | **`users` — Map** (core); optional scalar metadata bins (not fully enumerated in the cluster snapshot)                                                                                                            |
| **CDT usage**         | Map with **numeric keys** (MAPKEYS index type); multiple map keys per record (~**22–24M** SI entries vs **~19M** objects per node ⇒ several user ids per browser record)                                          |
| **TTL**               | Per-record TTL; cluster distribution **~31 days** at p100 (`default-ttl 0` in config)                                                                                                                             |
| **Secondary indexes** | **`browsers_users_key`** on bin **`users`**, index type **MAPKEYS**, numeric — used for query by user id across browser records                                                                                   |

#### Write operations

| Attribute                   | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**         | Production path is **UDF-heavy** (~**179k UDF/s** cluster-wide vs ~**9k** client writes/s and ~**63k** reads/s in the 2025-02-28 latency window). Aerospike has recommended migrating hot paths to **filter expressions** and CDT/`operate()` where possible (2025-09-04 CST record). Exact UDF names and whether they use Style (a) CDT ops vs Style (b) full-map replace are **not** in the cluster snapshot — treat as **Style (a) or UDF-wrapped map updates** until confirmed. |
| **Bytes written per write** | Small logical change per map key (typical C); full **~300–500 B** record rewrite on storage                                                                                                                                                                                                                                                                                                                                                                                         |
| **Bins modified per write** | Typically **`users`** (1 CDT bin)                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Concurrency control**     | Generation used today for cookie validation — **discouraged**; CST documents **server-side session record** with opaque `session_id` and fields such as `uid`, `exp`, `device_hash`, `last_seen`, `risk_state` as the target pattern (may apply to `user-primary` more than `browsers-primary`)                                                                                                                                                                                     |
| **Write frequency**         | ~**9k** client writes/s + **~179k** UDF/s; writes **skewed by node** (hot nodes up to **~3k** writes/s vs **<1k** on peers)                                                                                                                                                                                                                                                                                                                                                         |
| **Churn**                   | **~265M** expirations vs **~90M** live masters — steady replace/expiry workload                                                                                                                                                                                                                                                                                                                                                                                                     |

#### Read operations

| Attribute           | Value                                                                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Read operation**  | **`get()`** / batch reads; **secondary-index query** on **`users`** MAPKEYS (numeric user id); **~11k** query ops/s cluster-wide; legacy **UDF** on read path (~**72%** of ops in the snapshot window) |
| **Bytes returned**  | Full record (~**300–600 B** typical) or map subset via UDF/CDT projection                                                                                                                              |
| **Read frequency**  | ~**63k** reads/s; SLO stated as **P99 <10 ms**, **P99.9 <25 ms** (TAM intro record)                                                                                                                    |
| **Read:write (KV)** | ~**7:1** (63k : 9k). Including UDF as mutation traffic: **~1 read per ~3 UDF/write ops**                                                                                                               |

#### Scale and deployment context

| Attribute              | Value                                                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Cluster**            | 6 × nodes, **C-4.3.1.4**, **RF=2**, device `/dev/md2`, **128K** `write-block-size`                                                |
| **Namespace masters**  | **90.165M** (all sets); **`browsers-primary` ~57.9M** masters                                                                     |
| **Unique data**        | **26.031 GiB** on disk; **0 B** in-memory data (indexes in **50G** `memory-size` budget per node)                                 |
| **Client connections** | ~**15.6k** per node                                                                                                               |
| **Migration target**   | EE **7.2+** / **8.x** consolidated cluster; strong consistency and XDR discussed for session namespaces (2025-07 workshop record) |

#### Open items

- Map value schema inside **`users`** (nested map vs scalars per key).
- Which **UDF** modules touch **`browsers-primary`** / **`users`** vs other sets.
- Post-migration split: **`browsers-primary`** (C) vs **`user-primary`** / **`rtid-primary`** (likely **B**) in the same namespace.

#### What makes this customer distinct within Archetype C

- **Sub-KiB** map records (**~300–500 B**), not the 1–10 KiB “document” default in the summary matrix — same archetype, different size band.
- **MAPKEYS** SI turns “find all browser records for user X” into index-backed queries over map keys, not just primary-key `get`.
- **UDF dominates ops/sec** even though the **data model is CDT**; benchmarks must include UDF cost or an explicit “post–expression-migration” target profile.
- One namespace hosts **multiple archetypes**; **`browsers-primary` / `users`** should be weighted **~64%** in mixed session-store benchmarks.

---

## Archetype G: Segment Map Store (Single K-Ordered Map with Element-Level Expiry)

One record per entity holding a single K-ordered map bin with potentially thousands of entries. Each map entry contains embedded expiry metadata, allowing application-level per-element TTL within a single long-lived record.

### Representative Use Cases

- Ad-tech audience segmentation (user → thousands of interest segments with per-segment TTL)
- Feature flag stores (tenant → map of flag states with refresh times)
- Permission/capability caches (principal → map of capabilities with expiry)
- Shopping behavior profiles (user → map of category affinities with decay)

### Record Structure

| Attribute         | Value                                                                                                |
| ----------------- | ---------------------------------------------------------------------------------------------------- |
| **Key**           | Entity ID (user/device/cookie ID)                                                                    |
| **Record size**   | Few KiB to tens of KiB (scales with entry count; 1000 segments × ~20 bytes = ~20 KiB)                |
| **Bin count**     | 1                                                                                                    |
| **Bin types**     | K-ordered Map; key = segment_id (integer); value = List `[ttl_as_hours_since_epoch, attributes_map]` |
| **CDT usage**     | Single K-ordered map with list values containing optional nested maps                                |
| **Nesting depth** | 2 (map → list → optional attributes map)                                                             |

**Expiry encoding:** The TTL is stored as hours-since-a-fixed-epoch (not a wall-clock timestamp). This compact integer encoding supports `map_get_by_value_range` for "all non-expired segments" and `map_remove_by_value_range` for trim. Record-level TTL cannot serve this use case because different segments expire at different times.

### Write Operations

| Attribute                                | Value                                                                                                               |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Write operation — add/update segment** | `operate()` with `map_put("u", segment_id, [ttl, attributes])`                                                      |
| **Bytes written per write**              | ~20 bytes of new/updated map entry. Full record rewritten on storage.                                               |
| **Bins modified per write**              | 1                                                                                                                   |
| **Concurrency control**                  | None needed — one event stream per user; map_put is idempotent (upsert)                                             |
| **Write frequency**                      | Moderate — segments are added/refreshed as user activity occurs (e.g., ad impression → add segment with 30-day TTL) |
| **Server-side computation on write**     | K-ordered map key lookup (O(log N)) + insert/update                                                                 |

| Attribute                          | Value                                                                                           |
| ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Write operation — batch upsert** | `operate()` with `map_put_items("u", {seg1: [ttl1, {}], seg2: [ttl2, {}], ...})`                |
| **Bytes written per write**        | Multiple entries (e.g., 10 segments × 20 bytes = 200 bytes of new data). Full record rewritten. |

| Attribute                                  | Value                                                                                              |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| **Write operation — trim expired entries** | `operate()` with `map_remove_by_value_range("u", [0, null], [current_hour, null])`                 |
| **Bytes written per write**                | Removes expired entries; full record rewritten (now smaller).                                      |
| **Frequency**                              | Periodic — via background scan (hourly/daily) across all records in the set, rate-limited per node |

### Read Operations

| Attribute          | Value                                                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| **Read operation** | `operate()` with `map_get_by_value_range("u", [current_hour, null], [infinity, null])` — returns only non-expired segments |
| **Bytes returned** | Subset of the map (only live segments)                                                                                     |
| **Read frequency** | High — profile lookups happen on every ad bid decision (<10ms latency required)                                            |

### What Makes This Archetype Distinct

- Per-element expiry is embedded in the map values — the application manages it, not the server's TTL mechanism.
- Writes are small per-entry upserts, but the full record (potentially tens of KiB) is rewritten on storage each time.
- The trim operation (remove expired entries) is a write that reduces record size. It runs as a background scan across the entire set, rate-limited to avoid cluster overload.
- One record per user avoids the PI explosion of one-record-per-segment (which would create trillions of records for a large user base).
- `PERSIST_INDEX` with `V_ORDERED` can be used for large maps to improve value-range operations from O(N+M) to O(log N+M).

---

## Archetype H: Score-Bucketed Leaderboard (Composite-Key Map)

KEY_ORDERED maps with composite keys that encode `score + entity_id`, distributed across multiple records by score range. Key order equals rank order by construction, eliminating the need for a value index.

### Representative Use Cases

- Gaming leaderboards (global and per-region score rankings)
- Sales performance boards (revenue rankings by rep)
- Competitive fitness/health challenges (step count rankings)
- Auction bid rankings (price-ordered bid lists)

### Record Structure

**Scoreboard records** (multiple records representing score ranges):

| Attribute       | Value                                                                                                                 |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Key**         | Score-range identifier (e.g., integer 0, 1, 2... where record 0 holds scores 0–24, record 1 holds scores 25–49, etc.) |
| **Record size** | Tunable by score-range width (target: Goldilocks band, e.g., 50–500 KiB per record)                                   |
| **Bin count**   | 1                                                                                                                     |
| **Bin types**   | K-ordered Map; key = composite string `"00482-000000001"` (zero-padded score + player_id); value = player_id string   |
| **CDT usage**   | Single K-ordered map — key order IS rank order within this score range                                                |

**Player records** (one per player, separate set):

| Attribute     | Value                          |
| ------------- | ------------------------------ |
| **Key**       | Player ID                      |
| **Bin count** | 5–8 (score, name, metadata)    |
| **Bin types** | Integer (score), Strings, etc. |

### Write Operations

| Attribute                                             | Value                                                                                                                         |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Write operation — score update (same score range)** | Single `operate()` on one scoreboard record: `map_remove_by_key(old_composite_key)` + `map_put(new_composite_key, player_id)` |
| **Bytes written per write**                           | Remove ~30 bytes + insert ~30 bytes. Full record rewritten on storage.                                                        |
| **Bins modified per write**                           | 1 (the map bin on the scoreboard record)                                                                                      |

| Attribute                                                         | Value                                                                                                                                                 |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation — score update (crosses score range boundary)** | Transaction: `operate()` on old scoreboard record (remove) + `operate()` on new scoreboard record (put) + `put()` on player record (update score bin) |
| **Records written**                                               | 3 (two scoreboard records + one player record)                                                                                                        |
| **Concurrency control**                                           | Transaction (Aerospike 8+, strong-consistency namespace) to keep scoreboard and player record in sync                                                 |
| **Write frequency**                                               | Moderate — as players complete games/activities, their scores change                                                                                  |
| **Server-side computation on write**                              | Map key lookup + removal (O(log N)); map key insertion (O(log N))                                                                                     |

### Read Operations

| Attribute                        | Value                                                                                                                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Read operation — rank window** | `operate()` with expression: `MapExp.getByKey(INDEX)` to find player's position, then `MapExp.getByIndexRange(start, count)` to get surrounding entries. Returned via `ExpOperation.read`. |
| **Read operation — top N**       | Read the highest-score-range record, `map_get_by_index_range(-N, N)` for the last N map entries (highest scores).                                                                          |
| **Bytes returned**               | Small — only the requested rank window (e.g., 20 entries × ~30 bytes = ~600 bytes)                                                                                                         |

### What Makes This Archetype Distinct

- The composite-key trick means map key order = score order — index position within the map IS rank. No separate rank computation.
- Score updates are a remove + insert (position change), not an in-place modification.
- Cross-range score changes require multi-record transactions because two different records must be updated atomically.
- Write amplification on the scoreboard record depends on how many players are in that score range — a full rewrite of 500 KiB to change one player's entry is possible.
- Sharding by score range distributes write load — popular score ranges can be further subdivided.

---

## SubMilliPost Platform-Scale Assumptions

The following traffic estimates for Archetypes E, D, I, and J are derived from the [SubMilliPost](https://github.com/aerospike/aerospike-submillipost) sizing reference (Substack analysis: 205K subscribers, 489–672 comments per post, 129–136 distinct commenters, 10–75 reposts per post) and standard social-platform engagement ratios.

| Parameter                           | Assumed Value     | Basis                                                              |
| ----------------------------------- | ----------------- | ------------------------------------------------------------------ |
| Total registered users              | 200K              | Rounded from 205K subscriber reference                             |
| Daily active users (DAU)            | 60K (30% DAU/MAU) | Typical newsletter-platform engagement                             |
| Content creators (post weekly+)     | 1,000 (0.5%)      | Substack creator-to-reader ratio                                   |
| Active note creators (note weekly+) | 10,000 (5%)       | Short-form is lower-friction than long-form                        |
| New posts/day                       | ~150              | 1,000 creators ÷ 7 days                                            |
| New notes/day                       | ~3,000            | 10K note creators, ~2/week average                                 |
| New comments/day                    | ~8,000            | Derived: ~150 active posts × ~50 comments/day during active window |
| Likes/day                           | ~30,000           | 20% of DAU × 2.5 likes each                                        |
| Follows/unfollows per day           | ~1,000            | ~1.7% of DAU                                                       |
| Reposts/day                         | ~1,500            | Sizing reference: 10–75 per post; ~10 per active post/day          |
| Feed loads/day                      | ~300,000          | DAU × 5 loads/day                                                  |
| Content views/day                   | ~180,000          | DAU × 3 individual post/note views                                 |
| Profile views/day                   | ~60,000           | DAU × 1/day                                                        |
| Notification checks/day             | ~180,000          | DAU × 3 checks/day                                                 |
| Avg subscriptions per user          | ~15               | Newsletter-style; lower than follow-heavy platforms                |

---

## Archetype E: N:M Association List Store

Dedicated single-bin records holding ordered lists of related entity keys. The record's sole purpose is to represent one side of a many-to-many relationship.

### Representative Use Cases

- Social follow/follower lists (user A follows users B, C, D...)
- Content like lists (post X liked by users A, B, C...)
- Playlist membership (playlist contains tracks 1, 2, 3...)
- Access control lists (resource R accessible by users A, B, C...)
- Customer-to-account ownership (N:M)

### Record Structure

| Attribute       | Value                                                                               |
| --------------- | ----------------------------------------------------------------------------------- |
| **Key**         | Owner entity ID (e.g., user handle for `user_following`)                            |
| **Record size** | 1–150 KiB (scales linearly with relationship count: entries × ~15 bytes per handle) |
| **Bin count**   | 1                                                                                   |
| **Bin types**   | Ordered List of strings (or integers), with `persistIndex`                          |
| **CDT usage**   | Single ordered list — the entire record payload                                     |

**Example (SubMilliPost `user_followers`):**

| Bin       | Type                                    | Size at p99                                    |
| --------- | --------------------------------------- | ---------------------------------------------- |
| `handles` | Ordered list of strings, `persistIndex` | ~150 KiB (10,000 followers × ~15 B per handle) |

### Write Operations

| Attribute                            | Value                                                                                                                             |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation — add entry**      | `operate()` with `list_append("handles", value, policy={ORDERED, ADD_UNIQUE, NO_FAIL})`                                           |
| **Bytes written per write**          | ~15 bytes of new data (one handle string) inserted into the list. Full record rewritten on storage.                               |
| **Bins modified per write**          | 1                                                                                                                                 |
| **Concurrency control**              | Atomic single-record. `ADD_UNIQUE` with `NO_FAIL` makes duplicate adds idempotent — no CAS needed.                                |
| **Write frequency**                  | Bursty — follows/likes happen in clusters around social events. Per-record: low for typical users, moderate for popular entities. |
| **Server-side computation on write** | Binary search (O(log N)) to find insertion point in the ordered list + uniqueness check                                           |

| Attribute                          | Value                                                     |
| ---------------------------------- | --------------------------------------------------------- |
| **Write operation — remove entry** | `operate()` with `list_remove_by_value("handles", value)` |
| **Bytes written per write**        | Removes ~15 bytes from the list. Full record rewritten.   |

**Bidirectional patterns:** A "follow" event writes to TWO records: `list_append` on the follower's `user_following` record AND `list_append` on the target's `user_followers` record. Each write is independent and atomic on its own record.

### Read Operations

| Attribute          | Value                                                                                                                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Read operation** | Paginated: `operate()` with `list_get_by_index_range("handles", -N, N)` for the latest N entries; or full `get()` for membership checks (e.g., feed filtering loads the full blocked list) |
| **Bytes returned** | Page: N × ~15 bytes. Full list: up to 150 KiB at p99.                                                                                                                                      |
| **Read frequency** | Higher than writes — the list is read on every feed load, profile view, or relationship display                                                                                            |

### What Makes This Archetype Distinct

- Each write adds or removes one element (~15 bytes), but the full record (potentially 150 KiB) is rewritten on storage. Write amplification scales with list size.
- `persistIndex` stores the offset index in the list particle on disk, keeping `ADD_UNIQUE` at O(log N) even as the list grows to thousands of entries.
- The record has no useful structure beyond the list — it exists purely to represent one direction of a relationship.
- High-skew workloads (celebrity with millions of followers) are the scaling challenge. Overflow triggers a companion record with shard-on-demand.

### Platform-Scale Traffic (SubMilliPost)

**Sets:** `user_following`, `user_followers`, `content_likes`

| Metric                                         | Daily Volume | Derivation                                                                         |
| ---------------------------------------------- | ------------ | ---------------------------------------------------------------------------------- |
| **Writes — follow/unfollow**                   | ~2,000       | 1,000 events × 2 records (follower's `user_following` + target's `user_followers`) |
| **Writes — content likes**                     | ~30,000      | Each like appends handle to `content_likes` record                                 |
| **Total writes/day**                           | **~32,000**  |                                                                                    |
| **Reads — feed construction**                  | ~300,000     | Every feed load reads viewer's `user_following` (1 read per feed load)             |
| **Reads — like-list checks**                   | ~180,000     | Content views check `content_likes` (display likers or "already liked" state)      |
| **Reads — profile follower/following display** | ~60,000      | Profile views paginate followers/following lists                                   |
| **Total reads/day**                            | **~540,000** |                                                                                    |
| **Read:Write ratio**                           | **~17:1**    | Dominated by feed-construction reads of `user_following`                           |

**Traffic shape:** Write load is distributed uniformly across user records (each follow event touches two distinct records). Read load is concentrated: every feed load reads the same `user_following` record for the viewer — power users who load feed frequently generate proportionally more reads on their own record.

---

## Archetype F: Time-Series Roll-Up (List-per-Time-Slice)

One record per entity per time slice (e.g., per day), containing a single list bin of compact tuples. Each record accumulates readings over its time slice via appends.

### Representative Use Cases

- IoT sensor telemetry (temperature, humidity, pressure per device per day)
- Application metrics (request latency samples per service per hour)
- Energy meter readings (power consumption per meter per billing period)
- Environmental monitoring (air quality readings per station per day)

### Record Structure

| Attribute       | Value                                                               |
| --------------- | ------------------------------------------------------------------- |
| **Key**         | Composite: `{entity_id}-{time_slice}` (e.g., `sensor42-2026-06-04`) |
| **Record size** | ~10 KiB for a full day (1440 tuples × ~7 bytes each)                |
| **Bin count**   | 1                                                                   |
| **Bin types**   | List of 2-element lists (tuples): `[time_offset_int, value_int]`    |
| **CDT usage**   | Single list of tuples                                               |

**Key encoding:** The time slice is part of the record key — the application computes the key from the entity ID + current date (or hour). This means one record per sensor per day; the application knows exactly which key to write to or read from.

**Value encoding:** Time is stored as minutes-since-midnight (0–1439), not a full epoch timestamp. Values are stored as integers with fixed-point scaling (e.g., temperature × 10 → 621 for 62.1°). This keeps each tuple to ~7 bytes in MessagePack.

### Write Operations

| Attribute                            | Value                                                                                                                         |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | `operate()` with `list_append("t", [minute, value])` or `list_append_items("t", [[m1,v1],[m2,v2],...])` for batched ingestion |
| **Bytes written per write**          | ~7 bytes of new data per reading appended. Full record rewritten on storage. At end of day, the record is ~10 KiB.            |
| **Bins modified per write**          | 1                                                                                                                             |
| **Concurrency control**              | None needed — each sensor writes to its own record (no contention)                                                            |
| **Write frequency**                  | Regular — one write per sensor per minute (or batched: one write per sensor every 5–15 minutes with multiple tuples)          |
| **Server-side computation on write** | List append (O(1) for unordered lists)                                                                                        |

**Growth pattern:** The record starts empty at midnight and grows linearly to ~10 KiB by end of day. Next day starts a new record (new key). Old records can have a TTL set for automatic retention management.

### Read Operations

| Attribute                                    | Value                                                                                        |
| -------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Read operation — one day**                 | `get(key)` — full record                                                                     |
| **Read operation — time range within a day** | `operate()` with `list_get_by_value_interval("t", [start_minute, null], [end_minute, null])` |
| **Read operation — multi-day**               | Batch `get()` across computed keys (e.g., 365 keys for one sensor, one year)                 |
| **Bytes returned**                           | ~10 KiB per day-record; subset for time-range queries                                        |
| **Read frequency**                           | Bursty — analysis/display reads are periodic, not continuous                                 |

### What Makes This Archetype Distinct

- Writes are append-only — data is never modified after being written. Each reading adds ~7 bytes to a growing list.
- The record grows predictably from 0 to ~10 KiB over one time slice. No unbounded growth because the time-slice boundary creates a new record.
- Write amplification increases through the day: the 1440th append rewrites the full 10 KiB record to add 7 bytes.
- Reads are efficient because the key design makes the key set computable — no secondary index or scan needed.
- One sensor writing to one record means zero write contention.

---

## Archetype D: Consolidated Hierarchy (Tree-in-a-Record)

A complete tree structure consolidated into a single record using deeply nested K-ordered maps, with auxiliary sort/index bins maintained atomically alongside the primary data in the same record.

### Representative Use Cases

- Comment threads on articles/posts (nested reply tree with rank and time ordering)
- Threaded email conversations (message tree with metadata)
- Organizational hierarchies (department/team/member tree)
- Bill-of-materials / part assemblies (component tree)

### Record Structure

| Attribute         | Value                                                                                                           |
| ----------------- | --------------------------------------------------------------------------------------------------------------- |
| **Key**           | Parent entity composite key (e.g., `post:550e8400-e29b-41d4-a716-446655440000`)                                 |
| **Record size**   | 50–350 KiB typical; can reach low-MiB range for popular content                                                 |
| **Bin count**     | 4–6                                                                                                             |
| **Bin types**     | Nested K-ordered maps (primary tree), ordered lists (time index), K-ordered map (rank index), integer (counter) |
| **CDT usage**     | Heavy — the record IS a nested CDT structure                                                                    |
| **Nesting depth** | 3–8 levels (tree depth: root map → comment map → replies map → nested replies...)                               |

Each reply level costs two CDT levels (a comment map and its `replies` map), so reply depth needs an explicit cap; see [cdt-api.md](cdt-api.md) § Depth contract.

**Bin layout (SubMilliPost `content_comments` example):**

| Bin             | Type                             | Size at p95 | Purpose                                                                                                                          |
| --------------- | -------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `tree`          | K-ordered map of maps (nested)   | ~250 KiB    | Primary comment data; keys = comment_ids, values = `{author, text, created_at_ms, edited, like_cnt, repost_cnt, replies: {...}}` |
| `comment_order` | Ordered list of strings          | ~10 KiB     | Time-ordered list of top-level comment IDs                                                                                       |
| `comment_rank`  | K-ordered map (comment_id → int) | ~10 KiB     | Score index for "most liked" retrieval                                                                                           |
| `comment_cnt`   | Integer                          | 8 B         | Total comment count across all levels                                                                                            |
| `commenters`    | Ordered list of strings          | ~2 KiB      | Distinct author handles (indexed for inverse lookups)                                                                            |

### Write Operations

| Attribute                            | Value                                                                                                                                                                                                                                                                           |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation**                  | Single `operate()` call containing multiple CDT operations across bins: `map_put` into `tree` (with nested context for the correct depth) + `list_append` to `comment_order` + `map_put` to `comment_rank` + `add` to `comment_cnt` + `list_append(ADD_UNIQUE)` to `commenters` |
| **Bytes written per write**          | One comment added: ~~350 bytes of new data inserted into the tree; plus ~16 bytes appended to order list; plus ~24 bytes to rank map; plus counter increment. Total new data: ~400 bytes. But the full record (~~250 KiB) is rewritten on storage.                              |
| **Bins modified per write**          | 4–5 (all auxiliary bins updated atomically with the tree)                                                                                                                                                                                                                       |
| **Concurrency control**              | Atomic single-record — all bin mutations in one `operate()` execute under the record lock                                                                                                                                                                                       |
| **Write frequency**                  | Low — comments trickle in over hours/days (p95 ~100 comments in the hottest 4-hour window for popular content)                                                                                                                                                                  |
| **Server-side computation on write** | CDT structure traversal to the correct nesting depth; map insertion; list append with ADD_UNIQUE dedup; counter arithmetic                                                                                                                                                      |

**Other write patterns on this record:**

- **Edit comment:** `operate()` with `map_put` at the correct nested context (overwrites `text` field + sets `edited = true`). ~300 bytes modified.
- **Like a comment:** `operate()` with `map_put` in the `tree` (incrementing `like_cnt` within the nested comment map) + `map_put` in `comment_rank` (updating the score). ~16 bytes modified.
- **Delete comment (author):** `operate()` replacing `text` with placeholder and `author` with "anonymous" at the correct nested context. ~100 bytes modified.
- **Delete subtree (content owner):** `operate()` with `map_remove_by_key` on the parent's `replies` + `list_remove_by_value` on `comment_order` + `map_remove_by_key` on `comment_rank` + decrement `comment_cnt`.

### Read Operations

| Attribute          | Value                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **Read operation** | Full `get()` — returns the entire record (~250 KiB) for client-side tree rendering                                 |
| **Read frequency** | Higher than writes (comment threads are read far more often than commented on); 70% of reads are full-tree fetches |

### What Makes This Archetype Distinct

- Writes are small (one comment = ~400 bytes of new data) but trigger a full record rewrite of ~250 KiB on storage. The write amplification ratio is high.
- All bins are updated atomically in one `operate()` — no multi-record coordination needed for adding a comment.
- The dominant read is a full `get()` that benefits from consolidation: one I/O returns everything needed to render a comment thread.
- The shard-on-demand pattern activates when the record approaches the operational threshold (low-MiB range).

### Platform-Scale Traffic (SubMilliPost)

**Set:** `content_comments`

| Metric                                      | Daily Volume | Derivation                                                                                                                                                    |
| ------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Writes — new comments**                   | ~8,000       | Each comment is a `map_put` into the tree + auxiliary bin updates in one `operate()`                                                                          |
| **Writes — comment likes**                  | ~12,000      | ~40% of platform likes target comments; each updates `like_cnt` in tree + `comment_rank` map                                                                  |
| **Writes — edits/deletes**                  | ~300         | Comment edits, author-deletes (placeholder), owner-deletes (subtree removal)                                                                                  |
| **Total writes/day**                        | **~20,300**  |                                                                                                                                                               |
| **Reads — content views with comment load** | ~90,000      | 50% of 180K content views load the comment thread (full `get()`)                                                                                              |
| **Reads — dedicated comment-thread views**  | ~60,000      | Direct navigation to discussion (scrolling, reply context)                                                                                                    |
| **Total reads/day**                         | **~150,000** |                                                                                                                                                               |
| **Read:Write ratio**                        | **~7:1**     | Comment threads are read more often than commented on, but the ratio is lower than feed-driven patterns because viewing comments requires explicit navigation |

**Traffic shape:** Write load is highly skewed — popular posts concentrate most comments. The top 5% of content_comments records receive ~60% of daily writes. Read load is also skewed (popular threads are viewed more) but less extremely so. Writes are individually small (~400 bytes of new data) but rewrite the full record (50–250 KiB at p95).

---

## Archetype I: Event Timeline (List-of-Structs with Record TTL)

Records holding an unordered list of structured maps, where the record key encodes entity + date so that each record represents one day's events. Record-level TTL handles automatic expiry.

### Representative Use Cases

- Notification feeds (like/comment/mention events per user per day)
- Activity logs (user actions per day with auto-expiry)
- Alert streams (security/monitoring alerts per entity per day)
- Audit trails with retention policies (events auto-expire after N days)

### Record Structure

| Attribute         | Value                                                                       |
| ----------------- | --------------------------------------------------------------------------- |
| **Key**           | Composite: `{entity_id}:{YYYY-MM-DD}` (e.g., `alice:2026-06-04`)            |
| **Record size**   | Up to ~40 KiB at p99 per day-record                                         |
| **Bin count**     | 1                                                                           |
| **Bin types**     | Unordered List of K-ordered Maps                                            |
| **CDT usage**     | List of maps (list-of-structs pattern)                                      |
| **Nesting depth** | 2 (list → map → scalar values)                                              |
| **TTL**           | Set on first write — e.g., 15 days. Record auto-expires; no cleanup needed. |

**Map element structure (each notification):** `{id: UUIDv4, type: "like"|"repost"|"comment", actor: handle, target_type: string, target_id: string, root_type: string|null, root_id: string|null, read: bool, visible: bool, ts: int64}` — approximately 160 bytes per element.

### Write Operations

| Attribute                            | Value                                                                                            |
| ------------------------------------ | ------------------------------------------------------------------------------------------------ |
| **Write operation — new event**      | `operate()` with `list_append("items", {id: ..., type: ..., actor: ..., ...})`                   |
| **Bytes written per write**          | ~160 bytes of new data (one map element) appended to the list. Full record rewritten on storage. |
| **Bins modified per write**          | 1                                                                                                |
| **Concurrency control**              | Atomic single-record. `ADD_UNIQUE` dedups the element maps whatever order the client sent.       |
| **Write frequency**                  | Moderate — bounded by social activity. At p99, ~200 events per user per day.                     |
| **Server-side computation on write** | List append (O(1) for unordered list)                                                            |

| Attribute                              | Value                                                                                                                |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Write operation — mark events read** | `operate()` with `modify_by_path("items", condition: read == false AND ts >= cutoff, modification: set read = true)` |
| **Bytes written per write**            | Modifies boolean fields on matching elements. Full record rewritten.                                                 |
| **Bins modified per write**            | 1                                                                                                                    |
| **Scope**                              | Can touch 1–14 records (across day boundaries)                                                                       |
| **Server-side computation**            | Path expression: iterate list, evaluate predicate per element, modify matches in place                               |

| Attribute                                         | Value                                                                                                                                         |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Write operation — block user (make invisible)** | `operate()` with `modify_by_path("items", condition: actor == blocked_handle, modification: set visible = false)` across up to 14 day-records |
| **Records written**                               | Up to 14 (one per day in the TTL window)                                                                                                      |
| **Server-side computation**                       | Path expression filtering + in-place modification on each matching element                                                                    |

### Read Operations

| Attribute          | Value                                                                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Read operation** | Multi-record: read today's record, then yesterday's, etc., using `select_by_path("items", visible == true)` on each, until N items collected |
| **Records read**   | 1–14 (depending on how many days back the client needs to look)                                                                              |
| **Bytes returned** | N × ~160 bytes (only visible, matching items)                                                                                                |

### What Makes This Archetype Distinct

- Writes are append-only for new events, but `modify_by_path` operations can modify many elements within the list in a single operation (batch state transitions).
- The key design (entity + date) means each record has a bounded maximum size (one day's events) and natural expiry via record TTL.
- No background cleanup is needed — records expire automatically.
- The "block user" operation demonstrates a write that touches up to 14 records, each requiring a full rewrite to flip boolean fields on matched elements.
- Write amplification: marking 3 notifications as "read" in a 40 KiB record rewrites the entire 40 KiB to storage.

### Platform-Scale Traffic (SubMilliPost)

**Set:** `user_notifications`

| Metric                               | Daily Volume | Derivation                                                                                                                                     |
| ------------------------------------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Writes — notification generation** | ~38,000      | Likes (30K) + comments-that-notify (~6.5K; not all comments notify the same user) + reposts (1.5K) → one `list_append` each                    |
| **Writes — mark-as-read**            | ~54,000      | 180K notification checks × 30% have unread items to mark → `modify_by_path` per day-record                                                     |
| **Writes — block-user cascades**     | ~700         | ~50 block events/day × up to 14 day-records each (set `visible = false` on actor's notifications)                                              |
| **Total writes/day**                 | **~93,000**  |                                                                                                                                                |
| **Reads — notification checks**      | ~300,000     | 180K checks × avg 1.7 day-records read per check (today + sometimes yesterday)                                                                 |
| **Total reads/day**                  | **~300,000** |                                                                                                                                                |
| **Read:Write ratio**                 | **~3:1**     | Relatively write-heavy compared to other patterns; every social interaction generates a notification write, and mark-as-read is itself a write |

**Traffic shape:** Write load (notification generation) is distributed across many users but temporally correlated — a popular post triggers bursts of likes/comments that each write to different users' notification records. Mark-as-read writes spike during peak engagement hours (correlated with notification checks). The 15-day TTL means ~900K active day-records exist at any time (60K DAU × 15 days), though most are cold after their first day.

---

## Archetype J: Multi-Bin Entity with Denormalized Counters and Reference Lists

Records with 8–12 bins mixing scalar metadata, denormalized counters, and short ordered reference-ID lists. This is the "workhorse" entity record for domain objects that need fast display reads and support multiple independent write operations.

### Representative Use Cases

- Social media user profiles (display info + follower/post counts + recent post ID lists)
- E-commerce product records (title, price, stock count, category, related SKU list)
- Content/article records (title, body, author, publish date, like count, tag list)
- Player/character records (name, level, score, inventory IDs, guild membership)

### Record Structure

| Attribute         | Value                                                                                                             |
| ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Key**           | Entity ID (handle, UUID, hash-based ID)                                                                           |
| **Record size**   | 5–30 KiB (driven by text fields and reference list sizes)                                                         |
| **Bin count**     | 8–12                                                                                                              |
| **Bin types**     | Mixed: strings (text content), integers (counters, timestamps), ordered lists of strings (reference IDs, handles) |
| **CDT usage**     | 2–4 ordered lists alongside scalar bins                                                                           |
| **Nesting depth** | 0–1 (flat lists only)                                                                                             |

**Bin layout (SubMilliPost `users` example):**

| Bin             | Type                             | Size            | Mutability                                              |
| --------------- | -------------------------------- | --------------- | ------------------------------------------------------- |
| `display_name`  | String                           | ~100 B          | Rarely written (user edits profile)                     |
| `post_ids`      | Ordered list of UUIDv4 strings   | ~7.2 KiB at p99 | Appended on post create; element removed on post delete |
| `note_ids`      | Ordered list of xxHash64 strings | ~16 KiB at p99  | Appended on note create; element removed on note delete |
| `post_cnt`      | Integer                          | 8 B             | Incremented/decremented on post create/delete           |
| `note_cnt`      | Integer                          | 8 B             | Incremented/decremented on note create/delete           |
| `following_cnt` | Integer                          | 8 B             | Incremented/decremented on subscribe/unsubscribe        |
| `follower_cnt`  | Integer                          | 8 B             | Incremented/decremented on subscribe/unsubscribe        |
| `blocked`       | Ordered list of handles          | ~3 KiB at p99   | ADD_UNIQUE on block; remove on unblock                  |
| `created_at_ms` | int64                            | 8 B             | Immutable (written once at create)                      |
| `status`        | String                           | ~10 B           | Written at create and on delete cascade start           |

**Total record size at p99:** ~26.5 KiB.

### Write Operations

Multiple different events trigger writes to this record, each touching different bins:

**Write 1 — User creates a post:**

| Attribute                   | Value                                                                                               |
| --------------------------- | --------------------------------------------------------------------------------------------------- |
| **Write operation**         | `operate()`: `list_append("post_ids", post_id)` + `add("post_cnt", 1)`                              |
| **Bytes written per write** | ~36 bytes new data (one UUID string + counter increment). Full ~26 KiB record rewritten on storage. |
| **Bins modified**           | 2 out of 10                                                                                         |

**Write 2 — Someone follows this user:**

| Attribute                   | Value                                                       |
| --------------------------- | ----------------------------------------------------------- |
| **Write operation**         | `operate()`: `add("follower_cnt", 1)`                       |
| **Bytes written per write** | 8 bytes modified. Full ~26 KiB record rewritten on storage. |
| **Bins modified**           | 1 out of 10                                                 |

**Write 3 — User blocks someone:**

| Attribute                   | Value                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------ |
| **Write operation**         | `operate()`: `list_append("blocked", handle, policy={ORDERED, ADD_UNIQUE, NO_FAIL})` |
| **Bytes written per write** | ~15 bytes new data (one handle). Full record rewritten.                              |
| **Bins modified**           | 1 out of 10                                                                          |

**Write 4 — User edits display name:**

| Attribute                   | Value                                         |
| --------------------------- | --------------------------------------------- |
| **Write operation**         | `operate()`: `put("display_name", new_value)` |
| **Bytes written per write** | ~100 bytes. Full record rewritten.            |
| **Bins modified**           | 1 out of 10                                   |

| Attribute                            | Value                                                                                                                          |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **Concurrency control**              | Atomic single-record. Different events write different bins — no CAS needed because the record lock serializes all operations. |
| **Write frequency**                  | Low per-entity — p95 ~20 writes/day across all write types combined                                                            |
| **Server-side computation on write** | Varies: arithmetic (counters), list append with binary search for uniqueness (blocked), simple bin put (display_name)          |

### Read Operations

| Attribute          | Value                                                                                                                         |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| **Read operation** | Full `get()` for profile rendering; or `operate()` with `list_get_by_index_range("post_ids", -10, 10)` for paginated timeline |
| **Bytes returned** | Full record: ~26 KiB. Paginated subset: ~360 bytes (10 post IDs).                                                             |
| **Read frequency** | Much higher than writes (profile views, feed construction)                                                                    |

### What Makes This Archetype Distinct

- Multiple independent write paths modify different bins on the same record — a "follow" event touches only `follower_cnt`, a "post" event touches only `post_ids` + `post_cnt`. They never conflict.
- Every write (even an 8-byte counter increment) triggers a full ~26 KiB record rewrite on storage.
- Denormalized counters exist to avoid computing list lengths on every read — the counter is maintained by the write path.
- Reference-ID lists enable the pattern: read user → get IDs → batch GET the referenced records. No secondary index scatter query needed.
- Overflow guardrails: when a reference list (e.g., `note_ids`) exceeds ~5,000 entries, a companion record takes over via shard-on-demand.

### Platform-Scale Traffic (SubMilliPost)

**Sets:** `users`, `posts`, `notes`

| Metric                                            | Daily Volume    | Derivation                                                                                                                                                                                                       |
| ------------------------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Writes — user records**                         | ~5,500          | Post creates (150; append `post_ids` + inc `post_cnt`) + note creates (3K; append `note_ids` + inc `note_cnt`) + follows (1K × 2 records for `follower_cnt`/`following_cnt`) + blocks (50) + profile edits (200) |
| **Writes — post records**                         | ~11,200         | Creates (150) + edits (50) + like-count increments (~10K; ~33% of 30K likes target posts) + repost-count increments (~1K)                                                                                        |
| **Writes — note records**                         | ~23,600         | Creates (3K) + edits (100) + like-count increments (~20K; ~67% of likes target notes) + repost-count increments (~500)                                                                                           |
| **Total writes/day**                              | **~40,300**     |                                                                                                                                                                                                                  |
| **Reads — feed construction (user records)**      | ~4,500,000      | 300K feed loads × 15 subscriptions → batch read 15 user records per feed (for `post_ids`, `note_ids` bins)                                                                                                       |
| **Reads — feed construction (post/note records)** | ~6,000,000      | 300K feed loads × ~20 content items displayed → batch read posts and notes                                                                                                                                       |
| **Reads — direct content views**                  | ~180,000        | Individual post/note page loads                                                                                                                                                                                  |
| **Reads — profile views**                         | ~60,000         | Profile page loads (full user record)                                                                                                                                                                            |
| **Total reads/day**                               | **~10,740,000** | Dominated by batch reads during feed construction                                                                                                                                                                |
| **Read:Write ratio**                              | **~267:1**      | Extremely read-heavy; validates PRD's "feed read latency prioritized over write latency"                                                                                                                         |

**Traffic shape:** Reads are overwhelmingly batch operations — 300K feed loads drive 10.5M record reads, but these are served as ~300K batch-get calls (each fetching 15–20 keys). Individual records are read many times per day: a user with 1,000 followers has their record batch-read ~5,000 times/day (1,000 followers × 5 feed loads). Write load on user records is low and uniform (~20/day at p95). Write load on content records is extremely skewed — a viral note may receive thousands of likes (and thus thousands of counter-increment writes) while most notes receive <10.

---

## Cross-Reference: Archetype Sources

| Archetype                 | Primary Source                                                                                                                                              | Secondary Reference                                                               |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| A1: Blob Store            | Industry pattern                                                                                                                                            | —                                                                                 |
| A2: Counters              | Industry pattern                                                                                                                                            | [concepts-and-patterns.md](concepts-and-patterns.md) § Shard-on-demand pattern    |
| B: Multi-Bin Scalar       | Industry pattern; telco session-data engagement (CPU, memory/SI and lab records); identity/trust platform B† (Trust, Key, Search cluster records)           | [concepts-and-patterns.md](concepts-and-patterns.md) § Small independent entities |
| B2: Extreme Multi-Bin     | Internal support case — large-scale web platform                                                                                                            | —                                                                                 |
| C: Document/CDT           | Industry pattern; web portal session store `browsers-primary` / `users` map (TAM intro, 2025-06-11 and 2025-09-04 CST records; 2025-02-28 cluster snapshot) | [cdt-api.md](cdt-api.md), [path-expressions.md](path-expressions.md)              |
| G: Segment Map            | [concepts-and-patterns.md](concepts-and-patterns.md) § Source 2 (User Profile Store)                                                                        | [cdt-api.md](cdt-api.md) § Map types                                              |
| H: Leaderboard            | [concepts-and-patterns.md](concepts-and-patterns.md) § Source 6 (Leaderboards)                                                                              | [expressions.md](expressions.md)                                                  |
| E: Association Lists      | [SubMilliPost](https://github.com/aerospike/aerospike-submillipost) data model spec v3 §5b (`user_following`, `content_likes`)                              | [follow-relationship-scale.md](follow-relationship-scale.md)                      |
| F: Time-Series Roll-Up    | [concepts-and-patterns.md](concepts-and-patterns.md) § Source 1 (IoT Sensors)                                                                               | —                                                                                 |
| D: Consolidated Hierarchy | [SubMilliPost](https://github.com/aerospike/aerospike-submillipost) data model spec v3 §4b.3 (`content_comments`)                                           | [one-to-many-relationships.md](one-to-many-relationships.md)                      |
| I: Event Timeline         | [SubMilliPost](https://github.com/aerospike/aerospike-submillipost) data model spec v3 §6b (`user_notifications`)                                           | [path-expressions.md](path-expressions.md)                                        |
| J: Multi-Bin Entity       | [SubMilliPost](https://github.com/aerospike/aerospike-submillipost) data model spec v3 §3b, §4b.1, §4b.2 (`users`, `posts`, `notes`)                        | [concepts-and-patterns.md](concepts-and-patterns.md) § Foundational Concepts      |
