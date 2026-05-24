# Nhập điểm — Grade entry

**Menu path**: Quản lý điểm › Nhập điểm
**Roles**: admin · staff · viewer · teacher *(viewer and teacher are read-only — teachers see a "Tài khoản giáo viên chỉ được phép xem" banner, the grade cells render as readOnly+disabled, and the Nhập Excel / Lưu thay đổi / Hủy thay đổi buttons are disabled. Xuất excel stays enabled for everyone)*
**Source files**: [`frontend/src/resources/quanLyDiem/nhapDiem.tsx`](../../../../frontend/src/resources/quanLyDiem/nhapDiem.tsx) + companion folder `nhapDiem/`
**Related API endpoints**:
- [`GET /api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md)
- [`POST /api/diem`](../../../api/endpoints/diem/post.md), [`PATCH /api/diem/:id`](../../../api/endpoints/diem/patch.md)
- [`POST /api/diem/import`](../../../api/endpoints/diem/post-import.md)
- [`GET /api/diem/summary`](../../../api/endpoints/diem/get-summary.md)
- [`GET /api/bieu-mau/export?fillGrades=true`](../../../api/endpoints/bieu-mau/get-export.md) (Xuất Excel button)

**Related workflows**: [grade-entry-and-summary.md](../../../workflows/grade-entry-and-summary.md)

## When to use

Open this page to enter scores per student per subject as the cohort progresses, or to view the end-of-cohort summary at the end of training. The page has two tabs — one for raw score entry, one for the cohort-level totals.

## Layout

- Tab bar at the top: **Nhập điểm** | **Tổng kết cuối khóa**.
- Below each tab: a filter bar (varies per tab), then a table of students.
- Both tabs share the same filter UI for `khoa`/`daiDoi`/`truong`/`he`.

## Tabs

### Tab 1 — "Nhập điểm"

The default tab. For one selected `(khoa, daiDoi, mon)` triple, shows a table with one row per student and editable cells for the relevant grade components (`thuongXuyen`, `mieng`, `giuaHP`, `cuoiHP`). Note: the component uses `cuoiHP` as its local state field name; this maps to the API/DB field `hetHP`.

**What it does:**
- On filter change, calls `GET /api/sinh-vien?khoa=&daiDoi=` to get the student list, then prefetches the visible page's grades via `GET /api/diem?mon=…&maSV=<csv>` (comma-separated student codes). The backend accepts the CSV and returns `$in`-matched rows. Each row's grade subdoc populates the editable cells.
- **Lưu thay đổi** → `POST /api/diem` per dirty row. Backend's `pre('validate')` hook recomputes `tbMon` and the cell updates in place.
- **Hủy thay đổi** → reverts the table to the last fetched state and clears the cross-page dirty buffer. Disabled when nothing is unsaved.
- **Tìm kiếm** is disabled while there are unsaved edits — click **Lưu thay đổi** or **Hủy thay đổi** first so a fresh fetch can't silently discard work.
- **Nhập từ Excel** → opens an upload modal. Pick a workbook + select `(he, mon, khoa, donViLienKet)`, then submit. Frontend calls `POST /api/diem/import` and shows a result summary modal (`inserted`, `updated`, `skipped`, `errors`, `missingStudents`).

**What it shows:**
- One row per student (`maSV`, `hoTen`, `lop`, `daiDoi`).
- Editable cells for: `thuongXuyen`, `mieng` (Cao đẳng only), `giuaHP`, `cuoiHP` (the component's local state name for the API/DB field `hetHP`; a reader looking for `hetHP` in component state will find `cuoiHP` instead).
- Read-only `tbMon` cell that updates after save.

**Teacher behavior:** the filter dropdowns pre-fill with the teacher's `teacherScope` values via `useTeacherScope()`. A teacher cannot select a daiDoi outside their scope.

### Tab 2 — "Tổng kết cuối khóa"

End-of-cohort summary view. For a selected `khoa` (and optionally `daiDoi`/`truong`), shows one row per student with their average across all subjects in the chosen `he`.

**What it does:**
- Calls `GET /api/diem/summary?he=Đại học&khoa=…&daiDoi=…&truong=…`.
- Backend aggregates `SinhVien.diem[]` and returns one record per student with per-subject `tbMon` and an overall average.
- **Read-only.** No in-place editing on this tab.
- **Xuất Excel** button calls `GET /api/bieu-mau/export` with `(he, danhSach='Sổ điểm…', mon, khoa, daiDoi?, truong?, fillGrades=true)`. The `fillGrades=true` flag tells the backend to populate each student's component scores (`thuongXuyen`, `mieng`, `giuaHP`, `hetHP`, `tbMon`) from the stored `diem` array for the selected `mon`. Students with no `diem` entry for that `mon` get blank cells. This is the only call signature in the app that sets the flag — biểu mẫu in ấn omits it so its templates remain blank for hand-printing.

**What it shows:**
- One row per student in the selected scope.
- One column per subject (only the subjects of the chosen `he`).
- An "Overall" column with the aggregate average.
- **Pagination** above the table, identical to other pages (`TablePagination` + `usePagination`): default 50 rows per page, page-size options 10/25/50/100. The summary endpoint returns the full result set in one shot (no server-side paging), so paging is purely a render concern — switching pages doesn't re-fetch. The page resets to 1 automatically whenever the underlying result set changes (new fetch, filter applied, page-size change).

## Common tasks

### Enter a single student's score for one subject

1. Switch to **Nhập điểm**.
2. Filter `khoa`, `daiDoi`, `mon`.
3. Find the row, type into the cells.
4. Click **Save**. Watch `tbMon` recompute.

### Bulk-import grades from a workbook

1. Stay on **Nhập điểm**.
2. Click **Nhập từ Excel**.
3. Choose `he`, `mon`, `khoa`, `donViLienKet`, then pick the file. Submit.
4. Read the result modal: `inserted` and `updated` are successes; `missingStudents` lists `maSV`s outside scope; `errors` lists per-row issues.

### Export the end-of-cohort grade book

1. Switch to **Tổng kết cuối khóa**.
2. Pick `he` and `khoa`. Optionally narrow by `daiDoi`/`truong`.
3. Click **Xuất Excel**. The browser starts a download.

## Edge cases / gotchas

- **`mieng` is Cao đẳng only.** When `he === 'Đại học'`, the `mieng` column is hidden and submitting a value for it has no effect (the schema ignores it for the first 4 subjects).
- **`tbMon` is read-only and auto-computed.** Manually editing it via the API has no effect; the model's `pre('validate')` hook always recomputes from the raw components.
- **Save races.** If two staff users edit the same student × subject simultaneously, the second save wins; the first user's value is silently overwritten. There's no in-app locking.
- **Importer scope.** The Excel importer respects the caller's `allowedUnits`/`teacherScope`. Students whose `maSV` is in the workbook but outside the caller's scope land in `missingStudents`, not `inserted`.
- **Empty `mon` cells in the importer preserve previous values.** Don't blank a cell expecting the system to clear the score — leave it out entirely or use the per-cell save flow.
- **`-` in a grade cell = miễn thành phần (exempt from that exam).** Type `-` into any of `Thường xuyên` / `Mi ệng` / `Giữa HP` / `Cuối HP`; the cell stores it as-is and saves it to the backend as `null`. `Điểm TB` then recalibrates: only the non-dashed components count, and their weights are rescaled to sum to the full curriculum weight. Example (Đại học, 10% / 30% / 60%): exempt from Giữa HP → `Điểm TB = (0.10·TX + 0.60·CuốiHP) / 0.70`. The em-dash `—` is also accepted (useful when admins paste from formatted spreadsheets). This is distinct from `monMienHoc` (whole-subject exemption), which still hides every input cell for the row.
- **Tìm kiếm gated by dirty state.** Editing any cell sets the dirty flag; the Tìm kiếm button is disabled with a tooltip until the user either saves or clicks Hủy thay đổi. The previous behaviour silently dropped unsaved edits when re-searching.

