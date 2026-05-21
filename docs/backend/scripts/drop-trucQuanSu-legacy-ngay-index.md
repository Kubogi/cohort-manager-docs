# `drop-trucQuanSu-legacy-ngay-index.js`

**Source**: [backend/src/scripts/drop-trucQuanSu-legacy-ngay-index.js](../../../backend/src/scripts/drop-trucQuanSu-legacy-ngay-index.js)

## What it does

Drops the leftover `{ngay: 1}` unique index (auto-named `ngay_1`) from the `truc_quan_su` MongoDB collection.

## Why

The current schema uniques on the compound `{khoa: 1, ngay: 1}` — two Khóa can share the same date. An older schema version uniqued on `{ngay: 1}` alone. Mongoose's `autoIndex: true` only **adds** missing indexes; it never drops stale ones. So deployments that pre-date the schema flip carry both indexes, and the legacy single-field one silently blocks cross-Khóa imports: every row for the second Khóa collides with the first Khóa's dates and `insertMany({ordered:false})` reports them all as duplicates → "0 inserted".

## When to run

Once per environment, after deploying the fix and **before** the next cross-Khóa import. Idempotent — if the index is already absent, the script logs that and exits cleanly.

## How to run

```bash
cd backend
npm run migrate:drop-trucQuanSu-ngay-index
```

Sample output:

```
Connected to MongoDB

Indexes BEFORE:
  - _id_: {"_id":1}
  - khoa_1_ngay_1: {"khoa":1,"ngay":1} UNIQUE
  - ngay_1: {"ngay":1} UNIQUE

Dropping legacy index ngay_1…
  Dropped.

Indexes AFTER:
  - _id_: {"_id":1}
  - khoa_1_ngay_1: {"khoa":1,"ngay":1} UNIQUE

Done.
```

## Verification

After the script completes, importing the same Excel file under two different Khóa should both succeed (two rows per date, scoped per Khóa). Regression test: `backend/src/tests/trucQuanSu.legacyIndex.test.js`.
