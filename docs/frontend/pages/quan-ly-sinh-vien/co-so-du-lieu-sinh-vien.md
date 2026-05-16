# Cơ sở dữ liệu sinh viên — Student database

**Menu path**: Quản lý sinh viên › Cơ sở dữ liệu sinh viên

**Roles**: admin · staff · viewer *(not visible to teacher)*

**Source files**: [`frontend/src/resources/quanLySinhVien/coSoDuLieuSinhVien.tsx`](../../../../frontend/src/resources/quanLySinhVien/coSoDuLieuSinhVien.tsx) + companion folder `coSoDuLieu/`

**Related API endpoints**: [`/api/sinh-vien/*`](../../../api/README.md#students-apisinh-vien), [`POST /api/sinh-vien/import`](../../../api/endpoints/sinh-vien/post-import.md)

**Related workflows**: [`student-lifecycle.md`](../../../workflows/student-lifecycle.md), [`excel-import-students.md`](../../../workflows/excel-import-students.md)

## When to use

The canonical student CRUD surface. Use it whenever you need to create, view, edit, or delete a student record; bulk-import a roster; or apply advanced search.

## Layout

- Filter bar at the top: `khoa`, `đơn vị liên kết`, `đại đội`, `trạng thái`, `hệ`. The filters intersect; clearing one widens the result set.
- Search-by-keyword box (matches `maSV` / `hoTen` / `lop`).
- **Tìm kiếm nâng cao** button opens a modal with finer-grained filters (date-of-birth range, gender, nganh).
- Action buttons: **Thêm sinh viên** | **Import Excel**. Note: the button label is "Import Excel", not "Nhập từ Excel". There is no "Xuất Excel" button in the UI (the backend has a stub export endpoint but no corresponding frontend button).
- Table below: paginated, sortable.

## Common tasks

### Add a student

1. **Thêm sinh viên** opens a modal.
2. Fill `maSV`, `hoTen`, `cccd`, `ngaySinh`, `nganh`, `noiSinh`, `gioiTinh`, `soDienThoai`, `lop`, `khoa`, `daiDoi`, `truong`, `ngayNhapHoc`.
3. Save → `POST /api/sinh-vien`. Validator enforces `daiDoi.khoa` matches the body's `khoa`.

### Bulk-import a class

See [`workflows/excel-import-students.md`](../../../workflows/excel-import-students.md). TL;DR: multi-sheet `.xlsx`, sheet name = `daiDoi`, header row 5, data row 6+.

### Find a student by name when you don't know their khoa

Type the name fragment in the search box, then expand with **Tìm kiếm nâng cao** if too many hits.

### Withdraw a student

Don't delete. Create a `quyết định` with `loaiQD: "Thôi học"` and process it. See [`workflows/decision-processing.md`](../../../workflows/decision-processing.md).

## Edge cases / gotchas

- **Per-unit scoping.** Staff with `allowedUnits` only sees students inside their scope. The filter dropdowns still show all units but yield zero results when out-of-scope.
- **Duplicate detection** is on `(maSV, hoTen, ngaySinh)` — same maSV with a slightly different name passes uniqueness. Audit periodically.
- **Deleting a student** physically removes the row but does **not** automatically delete their decisions or health records. Prefer status changes via QĐ.
