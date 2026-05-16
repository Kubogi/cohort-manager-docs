# ADR 0005 — Embed `diem` as a subdocument inside `SinhVien` (no separate Grade collection)

**Status**: Accepted
**Date**: ~2024 (recorded retroactively 2026-05-16)

## Context

The system tracks grades (điểm) for each student across a fixed set of 7 military-training subjects. Each grade record has four raw components (`thuongXuyen`, `mieng`, `giuaHP`, `hetHP`) plus a computed `tbMon` average.

The natural relational shape would be a top-level `Grade` (or `Diem`) collection with a foreign key to `SinhVien`. The actual choice here was to embed: `SinhVien.diem: [DiemSubdocSchema]`.

## Decision

Define `diem` as an **embedded array of subdocuments** on the `SinhVien` model. The subdocument has its own schema with:

- Field validation (`thuongXuyen ∈ [0, 10]`, `mon ∈ MON_ENUM`).
- A `pre('validate')` hook that recomputes `tbMon` from the raw components using integer arithmetic (avoids IEEE-754 drift).
- Mongoose-assigned `_id` per subdocument, so individual grade entries can be addressed by ID via the `/api/diem/:id` REST endpoints.

The REST endpoint paths `/api/diem/*` are a **logical view** over `SinhVien.diem[]`. There is no `diem` collection in MongoDB.

## Consequences

### Positive

- **Reads are always per-student.** The score-entry UI loads one student's full grade history with `SinhVien.findById(...)`. No join, no second query.
- **Writes are atomic per student.** Updating multiple grades for one student is a single `findOneAndUpdate` — no need for transactions.
- **Document size stays bounded.** With at most 7 subjects × ~10 fields each, a `SinhVien` document is at most a few KB. Well under Mongo's 16 MB document limit.
- **The auto-compute hook lives next to the data.** `tbMon` is *always* consistent with the raw scores because the hook fires on every save.

### Negative

- **Cross-student queries on grades are awkward.** "All students with `tbMon < 5` in `Quân sự chung`" requires `SinhVien.find({ 'diem.mon': 'Quân sự chung', 'diem.tbMon': { $lt: 5 } })` — which returns students where *some* `diem` matches, then post-filtering in JS to find the *exact* matching subdocument. Aggregation pipelines work too but are more code.
- **No global index on grade values.** You can't put a unique index on `(student, mon)` to enforce one grade per subject per student; you have to do it in application code (the controller checks for an existing subdocument before pushing).
- **Per-grade addressing is by Mongo `_id`.** The `/api/diem/:id` endpoints take the subdocument's `_id`, which means clients have to remember and pass it. Less natural than `(studentId, mon)`.
- **The `/api/diem/import` endpoint has to work in terms of `(maSV, mon)` lookups** — it can't bulk-insert into a `diem` collection because there isn't one. The current implementation pre-loads the relevant `SinhVien` set, mutates `diem[]` in JS, then dispatches `save()` calls in parallel chunks.

## Alternatives considered

- **Separate `Diem` collection with foreign key.** Standard relational shape. Rejected because the queries that benefit from joins (cross-student grade analysis) are infrequent compared to the per-student operations.
- **One field per `(mon, component)` on `SinhVien`** — e.g. `tbMonQuanSu`, `tbMonChinhTri1`. Rejected because it doesn't generalize: adding a new subject means schema changes. The enum-driven subdocument approach handles new subjects with a one-line addition to `MON_ENUM`.
- **Hybrid: embedded for the "current" cohort, separated for historical.** Rejected as over-engineering for the current scale.

## When this decision might want revisiting

If any of these happen, reopen the design:

1. A new feature needs to **rank students across cohorts by grade in a specific subject**. The current shape can express it (with aggregation), but a separate collection with an index on `(mon, tbMon)` would be much faster.
2. Document size starts approaching 1 MB per student (e.g. someone adds an attachments-per-grade feature embedded in the subdocument).
3. Grade history per subject needs to be retained (not just current grades). The current schema would need an arbitrary-length history array, which would balloon student documents.

## See also

- [`/docs/architecture/database.md`](../architecture/database.md#5-embedded-vs-referenced--the-diem-choice) — full discussion
- [`/backend/src/models/sinhVien.js`](../../backend/src/models/sinhVien.js) — the schema source
- [`/docs/architecture/excel-pipeline.md`](../architecture/excel-pipeline.md#6-the-tbmon-arithmetic) — the integer-arithmetic computation for `tbMon`
- [`/docs/workflows/grade-entry-and-summary.md`](../workflows/grade-entry-and-summary.md) — how the embedding plays out in the score-entry flow
