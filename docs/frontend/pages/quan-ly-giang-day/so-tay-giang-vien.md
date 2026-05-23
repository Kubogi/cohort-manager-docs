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

**Subject columns reshape per Hệ** (mirrors `nhapDiem` "Tổng kết cuối khóa"). The hook routes the column set off `heDiem`:

| `heDiem` | Columns | Subjects matched on `grade.mon` |
|---|---|---|
| `""` (default) or `"Đại học"` | HP1, HP2, HP3, HP4 | Đường lối QP&AN của ĐCSVN, Công tác QP&AN, Quân sự chung, Kỹ thuật chiến đấu BB&CT |
| `"Cao đẳng"` | CT1, CT2, QS | Chính trị 1, Chính trị 2, Quân sự |

Without this branch the table always rendered 4 Đại học columns; Cao đẳng cohorts produced rows where every Học phần cell was blank but Điểm TBC + Xếp loại were populated (`allAverages` sums every `g.mon` regardless of column match). Switching `heDiem` re-derives `gradeRows` from the in-memory `grades` — no `/diem` refetch.

### Tab 3 — "Cập nhật hồ sơ sức khỏe"

Source: `CapNhatHoSoSucKhoeView.tsx`.

Health-record management. Filters: `khoa`, `daiDoi`, `truong`. **Writable for teachers within scope** — they can update a record's `thoiGianVe` and `trangThai` when a student is discharged, without going to the full [Hồ sơ sức khỏe page](../quan-ly-sinh-vien/ho-so-suc-khoe.md) (which is hidden from teachers anyway).

Each row joins the health record back to its student for the info columns (`Họ và tên`, `Đại đội`, `Số điện thoại`, `Trường`). The join goes through a merged `studentMap` — the SV-tab paginated students plus a separate `studentSupplements` map filled by an on-demand `getMany` for IDs the records reference but the SV-tab page doesn't contain. Without the supplement, SK records pointing at students outside the SV-tab page (a common case, because the two tabs have independent filters) would render with blank student-info cells.

**Column layout** mirrors the hoSoSucKhoe Thống kê table minus the Khoa column and the inline-edit actions (teachers are scoped to one khoa, and the edit modal lives elsewhere): STT, Họ và tên, Đại đội, Thầy quản lý, Khung (Chính trị / Quân sự), Số điện thoại, Trường, Thời gian đi (Giờ / Ngày), Thời gian về, Bệnh viện, Lý do, Chuẩn đoán BV, Người đi kèm (Sinh viên / Người thân), Thuốc ĐTCC, Sinh viên đi cùng. The `Khung` and `Người đi kèm` 2-subcol cells render an `x` when `record.khung` / `record.nguoiDiKem` matches the subcol label. Sinh viên đi cùng is a free-text companion-name field (separate from the `Người đi kèm` enum).

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
