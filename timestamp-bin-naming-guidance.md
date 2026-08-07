# Aerospike Timestamp and Bin Naming Contract Guidance

Purpose: prevent cross-client schema drift by standardizing timestamp bin names, units, and mutability rules.

Use with:

- `concepts-and-patterns.md`
- `new-app-modeling-checklist.md`
- `id-selection-guidance.md` (for ID contract alignment)

---

## 1) Normative rules

- Every timestamp bin must have one canonical unit/type contract.
- Suffixes are semantic contracts, not style preferences.
- Do not mix `created_at` and `created_at_ms` for the same semantic field.
- Do not mix ISO-string timestamps and epoch-ms integers for the same semantic field.
- Timestamp mutability must be explicit (`immutable` or `mutable`) per bin.
- No implementation-ready spec without a timestamp/bin naming decision record.

---

## 2) Canonical naming convention (recommended default)

Use epoch milliseconds in signed integer bins as the default:

- `*_at_ms` for point-in-time timestamps (`created_at_ms`, `updated_at_ms`, `publish_date_ms`).
- `*_until_ms` for deadlines/expiry boundaries.
- `*_from_ms` / `*_to_ms` for bounded windows.

Interpretation contract:

- `_ms` means epoch milliseconds UTC as integer only.
- If `_ms` is present, string/ISO writes are invalid.
- If no `_ms` suffix is used for time bins, the spec must explicitly define the alternate format and why.

---

## 3) Bin naming contract

- Keep names descriptive, then abbreviate only when needed to satisfy Aerospike 15-character bin-name limit.
- If abbreviating, define canonical long-form meaning in the spec and do not alias multiple spellings.
- Preserve semantic symmetry where possible:
  - `created_at_ms` / `updated_at_ms`
  - `start_at_ms` / `end_at_ms`
- Avoid ambiguous names (`time`, `date`, `ts`) unless strictly scoped and documented.

### 3.1) Common abbreviation reference (15-char limit)

When a descriptive bin name exceeds 15 characters, use these recommended abbreviations for common fields. This table is intentionally non-exhaustive — it covers high-frequency patterns only. For project-specific abbreviations not listed here, define them in the spec's schema-normalization table.

| Long form | Abbreviated | Chars | Notes |
|---|---|---|---|
| `publish_date_ms` | `pub_date_ms` | 11 | |
| `created_at_ms` | `created_at_ms` | 13 | Fits; no abbreviation needed |
| `updated_at_ms` | `updated_at_ms` | 13 | Fits; no abbreviation needed |
| `last_modified_at_ms` | `last_mod_ms` | 11 | |
| `follower_count` | `follower_cnt` | 12 | |
| `notification_type` | `notif_type` | 10 | |
| `conversation_ids` | `convo_ids` | 9 | |
| `distinct_authors` | `dist_authors` | 12 | |
| `repost_source_type` | `repost_typ` | 10 | |
| `cleanup_state` | `cleanup_state` | 13 | Fits; no abbreviation needed |

Rule: if a model spec uses an abbreviation not in this table, define the long-form meaning in the spec's bin schema section. Do not alias multiple spellings for the same field across artifacts.

**ID-recipe variables vs bin names:** Terms like `created_at_canonical` that appear in ID hash recipes (see [id-selection-guidance.md](id-selection-guidance.md) section 6) are recipe variables, not bin names. They are not subject to the 15-character bin-name limit. Do not confuse them with the bin that stores the timestamp value (e.g., `created_at_ms`).

---

## 4) Timestamp mutability contract

For each timestamp bin, explicitly declare:

- Mutability: immutable or mutable.
- Write source of truth: server clock, client clock, or imported external source.
- Update trigger: create-only, every mutation, specific lifecycle event.

Recommended defaults:

- `created_at_ms`: immutable, set at create.
- `updated_at_ms`: mutable, set on mutation. Avoid if you can use the record's last-update-time (LUT) metadata.
- `publish_date_ms`: immutable unless product requirement says republish/editable publish date.

---

## 5) Serialization and timezone contract

- Epoch-millis integers are always UTC-based.
- If API accepts ISO input, normalize to epoch ms at service boundary.
- Define precision and rounding behavior (truncate or round) once.
- Reject timestamps with unit ambiguity (seconds vs milliseconds) at validation boundary.

---

## 6) Migration and compatibility rules

When changing timestamp format or bin name:

- Introduce new bin with explicit migration state; do not silently reinterpret old bin values.
- Define read precedence during migration (`new_bin` first, fallback `old_bin`).
- Define dual-write window and cutover criteria.
- Define completion condition for removing old bins.

Do not lock contract status until migration behavior is documented when any rename/type change exists.

---

## 7) Required decision template (per major timestamp field group)

Capture the following in the model decision record:

| Field | Required content |
|---|---|
| Semantic field | Example: created time, publish time, notification event time |
| Bin name | Canonical bin name |
| Type and unit | `int64 epoch_ms` or approved alternative |
| Mutability | immutable/mutable + trigger |
| Producer | server/client/imported |
| Validation rules | accepted range, precision, ambiguity checks |
| API representation | request/response format and conversion rule |
| Migration posture | none / dual-read / dual-write + cutoff |

Gate:

- If type/unit/mutability are not explicit for each major timestamp field, mark `BLOCKED_MISSING_INPUT`.

---

## 8) Project-family recommendation

- Standardize on epoch-millis integer bins with `_ms` suffix for model timestamps.
- Prefer:
  - `created_at_ms`
  - `updated_at_ms` (when needed)
  - `publish_date_ms`
- Avoid mixed contracts like `created_at` (string|int) in one artifact and `created_at_ms` (int) in another.

