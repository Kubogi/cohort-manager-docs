# Báo cáo — Student reports

**Menu path**: Quản lý sinh viên › Báo cáo
**Roles**: admin · staff · viewer *(not visible to teacher)*
**Source files**: [`frontend/src/resources/quanLySinhVien/baoCao.tsx`](../../../../frontend/src/resources/quanLySinhVien/baoCao.tsx) + companion folder
**Related API endpoints**: [`GET /api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md), [`GET /api/bieu-mau/export`](../../../api/README.md#excel-form-export-apibieu-mau)

## When to use

Aggregate views over the student population — counts by status, by unit, by partner school. Useful for periodic reports to senior staff. Read-only on this page; the underlying data is edited elsewhere.

## Layout

- Filter bar: `khoa`, `đơn vị liên kết`, `đại đội`, `hệ`. No date range filter is implemented.
- Table of aggregated rows (one per chosen grouping dimension).
- **Xuất Excel** button to download.

## Common tasks

- **Quarterly headcount** — pick the khoa, leave other filters blank, view the table.
- **By partner school** — pick `truong`, optionally narrow further.
- **Excel export** — for inclusion in a printed report.

## Edge cases / gotchas

- **Scoping applies.** A staff user's `allowedUnits` filters the aggregates; the totals shown are *their* view, not global truth.
- **Cached data.** The hook caches results client-side until a filter changes. If someone edits students concurrently, refresh by changing a filter and back.
- **Row set follows the filter.** The table shows one row per đại đội whose metadata (khoa, đơn vị liên kết, ID) matches the active filter — đại đội outside the filter never appear, even with `quanSo: 0`. The "Thống kê theo khóa" rollup follows the same filtered subset.
- **Đơn vị liên kết lookup is bounded.** Schools (`don-vi-lien-ket`) are resolved via `dataProvider.getMany` using only the unique `donViLienKet` IDs referenced by the visible đại đội rows — not a paginated upfront fetch of the full catalog. This scales to large catalogs (hundreds of thousands of schools) without ever loading the whole list. Orphaned references (a `daiDoi.donViLienKet` pointing to a deleted school) fall back to the raw ID in the table rather than a blank cell.
- **Đại đội and khóa fetches use `perPage: 10000`** — large enough to cover the realistic operational scale of a few thousand đại đội / hundreds of khóa even on a 100k-student DB. The previous 200-row cap silently dropped rows past the first 200, making the table look truncated. For hundreds of thousands of đại đội, the aggregation should move server-side (flagged as future work).
- **`Số ca đi viện` counts every health record** for the đại đội regardless of `trangThai` — a student with three separate visits contributes three. The bucketing keys off the record's own denormalized `daiDoi` field (set at create/seed time), with a fallback to the student's current `daiDoi` only when the record's own value is missing. This means:
  - Records survive even when their `sinhVien` reference is dangling (deleted student) or the student isn't in the active students fetch.
  - Transferred students still show their old records under the đại đội where the visit was recorded, not their new đại đội.
  - Earlier versions iterated STUDENTS (group by current daiDoi, look up records by `student.id`) and silently dropped any record whose owner was missing/unscoped/lacked `daiDoi`. Production reproduction had ~240 records collapsing to ~57 in the rollup before this fix.
