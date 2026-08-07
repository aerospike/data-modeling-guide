# Aerospike Identifier Selection Guidance

Purpose: standardize how to choose identifier formats for new Aerospike models and avoid silent divergence across artifacts.

Use this guidance with:

- `concepts-and-patterns.md`
- `new-app-modeling-checklist.md` (section 10.11)

---

## 1) Normative rules

- Do not choose one ID format globally for all entities by default.
- Choose ID format per entity/relationship based on access pattern, repetition pressure, and identity semantics.
- If an ID is repeated heavily inside hot records (for example map keys, path indexes, order lists, rank maps), prefer short fixed-width IDs.
- If identity is naturally derived from immutable fields and retry idempotency across clients matters, prefer deterministic IDs.
- If there is no stable immutable tuple for identity, prefer globally unique opaque IDs (`UUIDv4`, or `ULID`/`UUIDv7` if all clients agree).
- Deterministic hash IDs are not valid unless the contract specifies algorithm, seed, canonical input, output encoding/length, and collision behavior.
- No final data model is implementation-ready until each major ID has an explicit decision record.

---

## 1.1) Primary ID format decision: cleartext composite vs hash

The first decision for each entity ID is whether the components appear in cleartext or are hashed. This decision comes before choosing which hash algorithm to use.

**Cleartext composite** (e.g., `alice-1742468400000`): the ID components are directly visible. Human-readable, useful for debugging and CLI work, but variable length.

**Hash** (e.g., `a1b2c3d4e5f67890`): the ID components are fed into a deterministic hash function. Compact, fixed size (predictable for capacity planning), but not human-readable.

**Decision driver:** whether operational readability or size predictability is the higher priority for that entity. When the ID is repeated heavily in nested structures (list/map bins across many records), fixed-size hashed IDs make capacity math simpler and save space at scale. When the ID is primarily a record key with low repetition pressure, cleartext composites improve operational visibility at no meaningful cost.

**If hash is chosen**, proceed to section 2 to select which hash. Default to UUIDv4 unless explicitly told otherwise. xxHash64 is a valid optimization when repetition pressure justifies the space savings, but it should not require a separate clarifying question — the modeler can apply it based on repetition-pressure analysis and the stakeholder can object if they disagree.

**Clarifying question guidance:** When the stakeholder provides ID components (e.g., `author_handle-created_at_canonical`), the clarifying question should ask: "Should these components appear in cleartext (e.g., `alice-1742468400000`) or be hashed to a fixed-size ID?" Do not ask "which hash?" — that is a downstream optimization, not a requirements question.

---

## 1.5) Precedence rules

When project-specific stakeholder answers or contracts specify ID formats that differ from the family defaults in this document (section 6), the project-specific contracts take precedence. Family defaults serve as starting points for clarification, not as overrides of confirmed project requirements.

Precedence order:

1. Project stakeholder-confirmed ID contracts (highest).
2. Family defaults in this document (section 6).
3. Generic rules in this document (section 1).

When an override occurs, the spec must log the override explicitly — stating which family default was replaced, by what, and why.

---

## 2) Decision framework: UUID/ULID vs deterministic short hash

This section applies when the primary decision (section 1.1) chose hash. If cleartext composite was chosen for an entity, this section does not apply to that entity's ID.

### Use UUID/ULID when

- The ID is mostly a record key and appears infrequently in repeated list/map bins.
- No immutable natural tuple exists (or tuple is large/unstable).
- Operational simplicity and broad tooling compatibility are the main priority.
- Sortable IDs are beneficial (choose ULID/UUIDv7 only with a cross-client agreement).

### Use deterministic short hash (`xxHash64` 16-hex) when

- The ID appears many times in consolidated records and contributes materially to record growth.
- Identity is derived from immutable tuple fields available to all clients.
- Clients must recompute the same identifier for retries/idempotent writes.
- You can enforce a strict canonicalization contract across all writers.

---

## 3) Deterministic ID contract (mandatory fields)

For every deterministic ID, specify:

- Algorithm and exact variant (example: `xxHash64`).
- Seed (example: `0`).
- Canonical input string recipe and delimiter.
- Canonical timestamp format and precision (example: epoch milliseconds decimal string). If the contract uses `created_at_canonical` or similar placeholder terminology, the spec must resolve this to an explicit format (e.g., epoch milliseconds decimal string, ISO-8601 compact `YYYYMMDDTHHmmssSSSZ`) before the ID contract is implementation-ready. An unresolved placeholder is a `BLOCKED_MISSING_INPUT`.
- Output encoding and length (example: lowercase hex, 16 chars).
- Allowed character set.
- Collision policy and API behavior.
- Cross-client compatibility requirement (all clients must implement identical recipe).

If any field above is missing, mark status `BLOCKED_MISSING_INPUT`.

**Multi-valued ID components.** When an ID recipe component maps to a multi-valued field on the entity (e.g., `authors` is a list but the recipe calls for `author`), the contract must resolve it to a single canonical value and document that resolution. For example: "Use `authors[0]` (first author in the immutable list); the list order is set at creation and never changes." Do not silently pick a convention — surface it as a group-specific clarification question during the entity-group pass (see checklist section 5.9.1) and confirm with the stakeholder.

---

## 4) Collision policy contract (mandatory)

Every model must define one of these policies (or an equivalent explicit variant):

- Reject-on-collision with typed error and caller retry semantics.
- Existence-check then accept/retry with deterministic tie-breaker.
- Regenerate-on-collision (randomized only if deterministic identity is not required).

Minimum operational requirements:

- Define how collision events are detected.
- Define response code/error contract.
- Define logging/metric name and alert threshold.
- Define remediation runbook (manual or automated).

Note: for deterministic IDs derived from `(author, created_at_ms)`, also state posture for same-author same-millisecond events.

---

## 5) Per-entity ID decision template (required)

Fill once per major entity (`user`, `post`, `note`, `comment`, `notification`) and any high-cardinality edge IDs. The `Generation mode` field is determined by the primary fork in section 1.1 (cleartext composite vs hash); if hash, section 2 determines which hash.

| Field                        | Required content                                                                                 |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| Entity/relationship          | Name and where ID is used                                                                        |
| Generation mode              | `readable_composite` / `uuid_ulid` / `deterministic_hash` (see section 1.1 for the primary fork) |
| Immutable tuple availability | Yes/No + tuple fields                                                                            |
| Repetition pressure          | p95/p99 placements (record key, list bins, map keys, edge records)                               |
| Size impact                  | Estimated bytes contributed at p95/p99                                                           |
| Canonical format             | Recipe, delimiter, timestamp/unit, encoding, length                                              |
| Collision policy             | Detection + behavior + observability                                                             |
| Idempotency requirement      | Retry/recompute behavior needed?                                                                 |
| Cross-client lock            | Shared implementation constraints                                                                |
| Migration path               | How format could evolve safely                                                                   |

Gate:

- If repetition pressure or collision behavior is unspecified, do not lock the ID format.

---

## 6) Project-family canonical defaults

Use this baseline unless requirements or measured evidence force a change:

- `post_id`: globally unique opaque ID (`UUIDv4` default).
- `notification_id`: globally unique opaque ID (`UUIDv4` default).
- `note_id`: deterministic short hash (`xxHash64(author + "-" + created_at_canonical)` -> 16 lowercase hex).
- `comment_id`: deterministic short hash (`xxHash64(author + "-" + created_at_canonical)` -> 16 lowercase hex).

**ID-recipe variable vs bin name:** `created_at_canonical` in the hash recipes above is an ID-recipe variable — the canonical string representation of the creation timestamp used as input to the hash function. It is not a bin name. The bin storing the creation timestamp has its own name within the 15-character bin limit (e.g., `created_at_ms`). The recipe variable and the bin name are separate concerns: the bin stores the value; the recipe defines how that value is serialized into the hash input string.

Rationale:

- `note_id` and `comment_id` are repeated heavily in nested comment and repost-related structures.
- `post_id` and `notification_id` have lower repetition pressure in hot nested bins.

---

## 7) Divergence guard for generated specs

When generating a draft model from partial inputs:

- If this guidance is in scope, do not emit `UUID/ULID` for all entity IDs as a default.
- If canonical ID rules are missing from inputs, emit `BLOCKED_MISSING_INPUT` for ID format and request clarification.
- Include an "ID Contract Assumptions" section in generated output that states chosen formats and why.
- If stakeholder answers confirm an ID format that differs from the family defaults in section 6, update the spec's ID contract to match the stakeholder-confirmed format and note the override. Do not silently fall back to the family default.
