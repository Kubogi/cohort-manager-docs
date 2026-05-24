# Hồ sơ sức khỏe — Health records

**Menu path**: Quản lý sinh viên › Hồ sơ sức khỏe
**Roles**: admin · staff · viewer · teacher *(teacher and viewer are read-only on most actions)*
**Source files**: [`frontend/src/resources/quanLySinhVien/hoSoSucKhoe.tsx`](../../../../frontend/src/resources/quanLySinhVien/hoSoSucKhoe.tsx) + companion folder `hoSoSucKhoe/` (HoSoListingPage, ThongKeDetailPage, BaoCaoAggregationPage)
**Related API endpoints**:
- [`GET /api/ho-so-suc-khoe`](../../../api/endpoints/ho-so-suc-khoe/get-list.md)
- [`POST /api/ho-so-suc-khoe`](../../../api/endpoints/ho-so-suc-khoe/post.md), [`PATCH /api/ho-so-suc-khoe/:id`](../../../api/endpoints/ho-so-suc-khoe/patch.md), [`DELETE /api/ho-so-suc-khoe/:id`](../../../api/endpoints/ho-so-suc-khoe/delete.md)
- [`GET /api/ho-so-suc-khoe/stats`](../../../api/endpoints/ho-so-suc-khoe/get-stats.md)
- [`POST /api/attachments`](../../../api/README.md#attachments-apiattachments) (for hospital-discharge files)

**Related workflows**: [`health-record-tracking.md`](../../../workflows/health-record-tracking.md)

## When to use

Track health events — illness, hospital admission, discharge — for individual students; review aggregate stats per khoa/đại đội; produce reports.

## Layout

- Tab bar at the top: **Hồ sơ sức khỏe** | **Thống kê** | **Báo cáo**.
- Each tab loads its own filter bar + table or summary view.

## Tabs

### Tab 1 — "Hồ sơ sức khỏe" (listing)

Source: `HoSoListingPage.tsx`.

**What it does:**
- Filter by `khoa`/`daiDoi`/`trangThai` (Bình thường / Viện). All four filters are **server-side**: `trangThai` is sent as a comma-separated `trangThai=` query param. When the filter is restrictive (one of the two values, not both), the hook bumps the records fetch to `perPage=10000` so the full match set arrives in one shot — Tab 1 then sources `displayedStudents` from the (now-complete) `healthRecords` via the supplemented `studentById` map, so every active-Viện student is visible regardless of which student page they live on. "Bình thường only" fires a second `perPage=10000` records fetch with `trangThai=Viện` under the same base + advanced filter scope to build `activeVienStudentIds`, then subtracts those IDs from the current student page. This correctly hides students whose mixed history (one closed Bình thường + one active Viện record) means they're currently in the hospital — the page-local pre-fix never subtracted because the primary records fetch only returned Bình thường rows, so the exclusion set was empty.
- **Pagination semantics under the Viện-only filter**: the table slices the FULL derived list client-side using `studentsPagination`. The footer total reads the derived count (e.g. "1–50 of 75"), not the server's student total. Paging the slice does NOT trigger a student refetch — the data is already complete in `healthRecords` + `studentById`. Clearing back to "both" reverts to server-paginated student fetching with the full student total in the footer.
- **Advanced student-field filters (`hoTen`, `maSV`, `lop`, `ngaySinh`) are forwarded to BOTH `/api/sinh-vien` and `/api/ho-so-suc-khoe`**. On the records endpoint, the backend resolves matching `SinhVien._id`s via a text-filter join and narrows the result set to `{ sinhVien: { $in: [...] } }`. Without this join the Viện-only branch in `derivedDisplayedStudents` would surface every Viện student in scope regardless of the user's name input — because the supplement effect would backfill every referenced student into `studentById`.
- **Supplement-failure visibility**: if the on-demand `getMany('coSoDuLieuSinhVien', { ids: missing })` fetch rejects OR returns fewer students than requested, the hook fires a `notify({ type: 'warning' })` reading "Không tải được thông tin sinh viên cho N/M hồ sơ". Previously the catch path was silent — a partial backend or transport hiccup made the Viện-only table go blank with no feedback.
- **Thêm hồ sơ** opens a modal — pick a student, then enter `thayQuanLy`, `khung` (Chính trị/Quân sự), `thoiGianDi.{gio, ngay}`, `benhVien`, `lyDo`, `chuanDoanBenhVien`, `nguoiDiKem` (Sinh viên/Người thân), `thuocDTCC`, `sinhVienDiCung`, `trangThai`.
- Optionally upload a discharge attachment via the `AttachmentField` (one file per record, owner type = `HoSoSucKhoe`).
- **Sửa** opens the same modal pre-filled. Common edit: set `thoiGianVe.{gio, ngay}` when the student is discharged, and flip `trangThai` from "Viện" back to "Bình thường".
- **Xoá** deletes the record (plus the attached file via the cascading service).

**What it shows:**
- One row per health record (the student can have multiple over time).
- Columns: student name, khoa, đại đội, trạng thái, ngày đi, ngày về, bệnh viện, lý do.
- The list mirrors the server-side student search verbatim. Students whose names are needed only to resolve the Thống kê tab's record→student join (e.g. health-record students past the page slice) are kept in a separate lookup map and **never appear in this table** — search results stay narrow.

### Tab 2 — "Thống kê"

Source: `ThongKeDetailPage.tsx`.

**What it does:**
- Per-record detail view with a richer filter set (by date range, by status, by `khung`).
- Inline editing of any row.

**What it shows:**
- More columns than Tab 1 — includes `khung`, `nguoiDiKem`, `chuanDoanBenhVien`, `sinhVienDiCung`.

**Student-info columns** (`Họ và tên`, `Khóa`, `Đại đội`, `Số điện thoại`, `Trường`) come from the hook's merged `studentById` map — the paginated students list plus on-demand `getMany` supplements for IDs referenced by visible records but missing from the current student page. This is why the columns render correctly even without any filters set; the previous lookup walked `data.students` (the current page only) and produced blank cells for any record whose student lived past the page slice.

### Tab 3 — "Báo cáo"

Source: `BaoCaoAggregationPage.tsx`.

**What it does:**
- Aggregated counts per `khoa` × `trangThai` × time window.
- Pulls data from `GET /api/ho-so-suc-khoe/stats`.

**What it shows:**
- A pivot-style table or chart with counts by `trangThai` per unit.
- **Số ca đi viện** is the count of *records* per đại đội, not the count of students with ≥1 record — a student with three separate health records contributes three to the column. With the default (both trangThai checked) the column reads as "total hospital visits per đại đội"; when the user narrows Tab 1's trangThai filter the records fetch is narrowed too, so the count reflects only the visible slice (e.g. Viện-only → currently-active visits per đại đội).
- **Đơn vị liên kết** is sourced from `dd.donViLienKet` on each `DaiDoi` lookup entry (the schema field is `donViLienKet`, not `truong` — a previous version of the hook read `dd.truong` and left the cell blank for every đại đội with zero records). The cell render falls back: `unitLookups.truongMap[id]` → on-demand `dataProvider.getMany('don-vi/don-vi-lien-ket', { ids: missing })` supplement → raw ID. Orphaned references therefore render the raw ID instead of going blank.

**Toolbar:**
- A **Xóa bộ lọc** button under the filter row resets `khoa`, `donViLienKet`, and `daiDoi` together — same handler as the sibling tabs. Disabled while data is loading.

**Pagination:** The stats table uses the shared `TablePagination` (default 50 rows, options 10/25/50/100). Pagination is **client-side** because `stats` is a small bounded aggregation (one row per đại đội). The `STT` column shows the global rank across the full stats list (sorted by số ca đi viện desc), so the top đại đội remains `STT=1` even on page 2+. The page auto-resets to 1 when the row count shrinks below the current page (e.g. filter change).

## Common tasks

### Record a new hospital admission

1. Open **Tab 1 — Hồ sơ sức khỏe**.
2. Click **Thêm hồ sơ**.
3. Pick the student (`SearchableSelect` over the filtered SinhVien list).
4. Set `trangThai: "Viện"`, fill `thoiGianDi`, `benhVien`, `lyDo`. Optionally upload an admission scan.
5. Save.

### Mark a student as discharged

1. Find their existing "Viện" record (filter `trangThai = Viện`).
2. **Sửa** the row. Fill `thoiGianVe`. Optionally update `chuanDoanBenhVien` and `thuocDTCC` (drugs prescribed).
3. Flip `trangThai` to "Bình thường".
4. Save.

A confirmation modal appears summarising the auto-appended `ghiChuYTe` line: `Đi viện lần N: dd/mm`. The `dd/mm` is the **admission** date (`thoiGianDi.ngay`) — the line describes *when the student went to hospital* — not the discharge date.

### Delete a record (with ghi chú auto-cleanup)

When a record is deleted, the backend re-derives the student's `ghiChuYTe` discharge block from the surviving records. So deleting "lần 1" of two records renumbers the surviving line to "lần 1" (instead of leaving an orphan "lần 2"). Deleting the last remaining record clears the discharge block entirely. User-authored notes that don't match the `Đi viện lần N: dd/mm` pattern are preserved verbatim — only the auto-generated lines are touched. See [`backend/src/utils/ghiChuYTeSync.js`](../../../../backend/src/utils/ghiChuYTeSync.js).

### Check how many students are currently in hospital

Switch to Tab 3 (Báo cáo) and look at the "Viện" count for the relevant unit.

## Edge cases / gotchas

- **One attachment per record.** The `Attachment` collection has a unique `{ ownerType, ownerId }` index, so each health record can only have one file. To swap, delete first then upload.
- **`thoiGianDi` and `thoiGianVe` are subdocuments** (`{ gio, ngay }`), not a single ISO datetime. The form combines them at save time; do not expect a single `Date` field.
- **`khung` enum is fixed**: `'Chính trị'` or `'Quân sự'`. Don't free-text it.
- **`trangThai` flip vs deletion.** Deleting a record removes the historical log; usually you want to *update* `thoiGianVe` and `trangThai` instead of deleting. When you do delete, the auto-generated `ghiChuYTe` discharge lines for that student are renumbered to match — see "Delete a record" above.
- **Mã SV is read-only in both add and edit forms.** The field is auto-filled from the row/record context (the Thống kê tab passes the student via the supplemented `studentById` map, so off-page students still get the correct Mã SV). Re-keying isn't supported — open the form via the right row instead.
- **No global cascade.** Deleting the underlying `SinhVien` does *not* automatically delete their health records (Mongo has no FK cascade). Prefer changing the student's status to `Thôi học` over deletion.
- **`HoSoSucKhoe` has no indexes** (per [HoSoSucKhoe schema](../../../backend/schemas/HoSoSucKhoe.md)). Queries scan. Acceptable today; revisit if row counts climb.
