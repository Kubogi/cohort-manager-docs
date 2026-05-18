# `backfill-student-sort-keys.js`

**Source**: [backend/src/scripts/backfill-student-sort-keys.js](../../../backend/src/scripts/backfill-student-sort-keys.js)

## What it does

One-time migration that populates `khoaSortKey` and `daiDoiSortKey` on every existing `SinhVien` document, and **`$unset`s the legacy `tenRieng` field** that the earlier Vietnamese-given-name sort needed (the within-DaiDoi sort is now `_id`, so the field is dead weight). New documents and updates take care of the two sort keys themselves via the Mongoose hooks on [SinhVien.js](../../../backend/src/models/SinhVien.js) — this script is for the rows that pre-date the current schema.

## When to run

- Once, immediately after deploying the schema change that adds the three denormalized sort keys.
- Optionally any time you suspect drift (e.g. a bulk update that bypassed the model layer). The script is idempotent — rows that already have the correct values are skipped.

## How to run

```bash
cd backend
node -e "import('./src/scripts/backfill-student-sort-keys.js')"
```

## Output

Per-1000-row progress lines plus a final summary:

```
…processed 1000 (updated 1000, skipped 0)
…processed 2000 (updated 1234, skipped 766)
Done: processed 2487, updated 1450, skipped 1037.
Legacy `tenRieng` field is now removed where present.
```

- **processed** — total `SinhVien` rows scanned.
- **updated** — rows whose `khoaSortKey` / `daiDoiSortKey` differed from the computed values OR that still carried a `tenRieng` field (re-written via `bulkWrite` in batches of 500).
- **skipped** — rows where the sort keys already matched AND `tenRieng` is already absent.

## How it computes the keys

Helpers live in [`backend/src/utils/sortKeys.js`](../../../backend/src/utils/sortKeys.js):

- `khoaSortKey` — `parseKhoaSortKey(Khoa.ten)` — strip non-digits then `parseInt`. Missing → `Number.MAX_SAFE_INTEGER`.
- `daiDoiSortKey` — `parseDaiDoiSortKey(DaiDoi.ten)` — strip leading non-digit prefix (covers `c`/`C`), strip remaining non-digits, then `parseInt`. Missing → `Number.MAX_SAFE_INTEGER`.

The script loads every Khoa and DaiDoi into in-memory `Map<_id, ten>` maps before iterating students, so each row only needs the in-memory lookup — no extra database round-trips per student.

## Connection

Uses the primary Mongo connection from `env.mongoUri` ([`backend/src/config/env.js`](../../../backend/src/config/env.js)). Disconnects cleanly on success; exits with code 1 on error.

## Tests

Helper-level tests in [`backend/src/tests/sortKeys.test.js`](../../../backend/src/tests/sortKeys.test.js); hook-level integration in [`backend/src/tests/sinhVien.sortKeys.test.js`](../../../backend/src/tests/sinhVien.sortKeys.test.js). The script itself is uncovered — it is a one-shot migration and the underlying logic is the same as the hooks.
