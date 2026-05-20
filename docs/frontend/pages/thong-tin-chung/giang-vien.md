# Giảng viên — Instructor management

**Menu path**: Thông tin chung › Giảng viên
**Roles**: admin · staff · viewer *(not visible to teacher)*
**Source files**: [`frontend/src/resources/thongTinChung/giangVien/giangVien.tsx`](../../../../frontend/src/resources/thongTinChung/giangVien/giangVien.tsx) + companion folder
**Related API endpoints**: [`/api/can-bo-quan-ly/*`](../../../api/README.md#management-staff-apican-bo-quan-ly)

> The page label is "Giảng viên" (instructor) but the backend resource is `CanBoQuanLy` (management staff). The dataProvider's `resourcePathMap` translates `giangVien → can-bo-quan-ly`. See [glossary](../../../glossary.md#people) for the naming history.

## When to use

Maintain the roster of management staff / instructors. These records are referenced by `DaiDoi.canBo[]` for time-bounded battalion assignments and by `User.taiKhoan` if the staff member has a system login.

## Layout

- Filter bar: search by `hoTen`, `donViQL`, `capBac`, `soQD`.
- Action buttons: **Thêm giảng viên** | **Nhập từ Excel** | **Xuất Excel**.
- Table: name, military rank, position, managing unit, decision number, validity, phone, linked account.

## Common tasks

- **Add an instructor** → modal with `hoTen` (required), `capBac`, `chucVu`, `donViQL`, `soDienThoai`, and optional linked `taiKhoan` (User). Note: `soQD`, `ngayRaQD`, and `hieuLuc` belong to a separate decision-history sub-record and are not fields in the create/edit form.
- **Link to a user account** → pick from the User dropdown. The dropdown lists every existing User account (admin-only `GET /api/users` populates the options) — not just users already linked to a CBQL. The `(taiKhoan)` partial unique index allows multiple rows with no link but at most one row per linked User. **Side-effect:** if the chosen User has `role: 'teacher'`, the system automatically syncs that user's `teacherScopeSynced[]` from this CBQL's `phanCong[]` — see [`docs/workflows/teacher-scope-sync.md`](../../../workflows/teacher-scope-sync.md).
- **Bulk import** → upload `backend/forms/Danh sách GV.xlsx` (the import template).
- **Delete** → confirmation; CanBoQuanLy is referenced from `DaiDoi.canBo[]`, so deletion may leave dangling references (no cascade).

## Edge cases / gotchas

- **Names are not unique.** Two staff with identical `hoTen` is allowed. Disambiguate by `soQD` or `donViQL`.
- **The "Giảng viên" name vs `CanBoQuanLy` collection** — they are the same record. The frontend hooks `useGiangVienData()` and `useKhoaHocData()` both read from `/api/can-bo-quan-ly`.
- **Deleting a staff member doesn't remove their references** in `DaiDoi.canBo[]`. Audit and clean up manually if needed.
