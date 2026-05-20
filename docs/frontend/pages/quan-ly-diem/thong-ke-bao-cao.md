# Thống kê, báo cáo — Grade statistics

**Menu path**: Quản lý điểm › Thống kê, báo cáo

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/quanLyDiem/thongKeBaoCao/thongKeBaoCao.tsx`](../../../../frontend/src/resources/quanLyDiem/thongKeBaoCao/thongKeBaoCao.tsx) + `ThongKeBaoCaoView.tsx`

**Related API endpoints**: [`GET /api/diem/summary`](../../../api/endpoints/diem/get-summary.md), [`GET /api/diem`](../../../api/endpoints/diem/get-list.md), [`GET /api/bieu-mau/export`](../../../api/README.md#excel-form-export-apibieu-mau)

## When to use

Aggregate grade statistics — pass/fail counts, distribution by subject, by `khoa` × `đại đội`. Use after a cohort completes their grades to produce reports.

## Layout

- Filter bar: `khoa`, `đại đội`, `truong`, `hệ`.
- Tabs or sub-views (depends on implementation) showing different aggregations:
  - Pass/fail counts by `mon`.
  - Histograms of `tbMon` per `mon`.
  - Per-`đại đội` averages.
- **Xuất Excel** for any view.

## Common tasks

- **End-of-cohort summary** — pick the khoa, view the aggregate. Compare with [Tổng kết cuối khóa](nhap-diem.md) for the per-student detail.
- **Excel report** — export to share with non-system users.

## Edge cases / gotchas

- **Scoping applies** — staff/teacher see only their accessible students' grades reflected in the aggregates.
- **Pass threshold is not codified in this app.** The Excel templates may render "Pass" / "Fail" but the system doesn't store a threshold. Communicate the rule externally.
- **Cached** — like Báo cáo, results are cached client-side per filter combination. Reload with a filter change to invalidate.
- **Columns**: Khóa · (Đơn vị liên kết · Đại đội on the per-đại-đội tab) · Xuất sắc · Giỏi · Khá · Trung bình · Nợ môn · **Tổng**. The `Tổng` column is the total student count for that khoa / khoa+đại đội group (sum of all bucket counts plus students with no grades yet and students whose `trangThai` suppresses the rollup such as `Học ghép` / `Học lẻ`).
