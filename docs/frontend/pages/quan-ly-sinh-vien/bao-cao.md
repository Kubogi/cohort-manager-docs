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
