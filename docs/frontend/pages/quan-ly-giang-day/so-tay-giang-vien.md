# Sổ tay giảng viên — Instructor handbook

**Menu path**: Quản lý giảng dạy › Sổ tay giảng viên

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/quanLyGiangDay/soTayGiangVien/`](../../../../frontend/src/resources/quanLyGiangDay/soTayGiangVien/) (parent file + 5 view components: `DanhSachSinhVienView`, `KetQuaDiemView`, `CapNhatHoSoSucKhoeView`, `NhanLichGiangDayView`, `ThongBaoView`)

**Related API endpoints**:
- [`GET /api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md)
- [`GET /api/diem/summary`](../../../api/endpoints/diem/get-summary.md)
- [`GET /api/ho-so-suc-khoe`](../../../api/endpoints/ho-so-suc-khoe/get-list.md), [`PATCH /api/ho-so-suc-khoe/:id`](../../../api/endpoints/ho-so-suc-khoe/patch.md)

## When to use

A dashboard for instructors (`teacher` role). Pulls together the four things an instructor needs day-to-day: student roster, grades, health updates, schedule. The viewer/staff/admin roles can also open it for read-only access.

## Layout

- Tab bar at the top with 5 tabs.
- Each tab has its own filter row (typically `khoa`/`daiDoi`/`truong`) and its own content panel.
- All filters auto-fill from `useTeacherScope()` for teachers — they see only their assigned units by default.

## Tabs

### Tab 1 — "Danh sách sinh viên"

Source: `DanhSachSinhVienView.tsx`.

Roster view. Filters: `khoa`, `daiDoi`, `truong`, `trangThai`. Advanced search by maSV / họ tên / ngày sinh.

Read-only for teachers. Staff can navigate to a student row but cannot create or delete students from here — those flows live on [Cơ sở dữ liệu sinh viên](../quan-ly-sinh-vien/co-so-du-lieu-sinh-vien.md).

### Tab 2 — "Kết quả điểm"

Source: `KetQuaDiemView.tsx`.

Grade summary by student and subject. Filters: `khoa`, `daiDoi`, `truong`, `he`, `trangThai`.

Read-only on this tab. To enter or edit grades, open [Nhập điểm](../quan-ly-diem/nhap-diem.md).

### Tab 3 — "Cập nhật hồ sơ sức khỏe"

Source: `CapNhatHoSoSucKhoeView.tsx`.

Health-record management. Filters: `khoa`, `daiDoi`, `truong`. **Writable for teachers within scope** — they can update a record's `thoiGianVe` and `trangThai` when a student is discharged, without going to the full [Hồ sơ sức khỏe page](../quan-ly-sinh-vien/ho-so-suc-khoe.md) (which is hidden from teachers anyway).

### Tab 4 — "Nhận lịch giảng dạy" — **stub**

Source: `NhanLichGiangDayView.tsx`. Placeholder; the teaching-schedule receiving flow is not implemented yet.

### Tab 5 — "Thông báo" — **stub**

Source: `ThongBaoView.tsx`. Placeholder; notifications are not implemented yet.

## Common tasks

### As a teacher, see today's roster for my battalion

1. Open the page. Tab 1 is the default.
2. Filters are pre-filled with your `teacherScope`. The table shows only your students.
3. Scan, search, or export.

### As a teacher, mark a student as discharged

1. Switch to **Tab 3 — Cập nhật hồ sơ sức khỏe**.
2. Find the student's "Viện" record.
3. Edit: fill `thoiGianVe`, change `trangThai` to "Bình thường", save.

### As staff, audit a teacher's reach

1. Open the page as a staff/admin user.
2. Change the filters away from the teacher's `teacherScope` to inspect what they *can't* see.
3. Cross-reference with the role-visibility matrix in [`docs/frontend/README.md`](../../README.md#role-visibility-matrix).

## Edge cases / gotchas

- **Two tabs are visually present but functionally stubs** (Tabs 4 and 5). Don't promise users those features work yet.
- **Teacher filters are stubborn.** `useTeacherScope()` re-applies its values when the page mounts. Switching tabs may reset filters to scope defaults. If a teacher is in two units, the singleton helpers pick the only one or leave the dropdown open.
- **Writes are scoped, not just reads.** Even though Tab 3 lets a teacher edit a health record, `applyTeacherScope` on the backend still enforces that the record belongs to one of their (khoa, đại đội) entries. A teacher who somehow loaded an out-of-scope record (link-sharing, browser back/forward) cannot save changes to it.
- **The Excel export on Tab 1/Tab 2 honors filter state.** Make sure the filters are the ones you want before clicking export.
