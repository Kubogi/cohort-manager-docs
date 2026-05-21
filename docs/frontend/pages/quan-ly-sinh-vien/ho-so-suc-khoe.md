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
- Filter by `khoa`/`daiDoi`/`trangThai` (Bình thường / Viện).
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

### Check how many students are currently in hospital

Switch to Tab 3 (Báo cáo) and look at the "Viện" count for the relevant unit.

## Edge cases / gotchas

- **One attachment per record.** The `Attachment` collection has a unique `{ ownerType, ownerId }` index, so each health record can only have one file. To swap, delete first then upload.
- **`thoiGianDi` and `thoiGianVe` are subdocuments** (`{ gio, ngay }`), not a single ISO datetime. The form combines them at save time; do not expect a single `Date` field.
- **`khung` enum is fixed**: `'Chính trị'` or `'Quân sự'`. Don't free-text it.
- **`trangThai` flip vs deletion.** Deleting a record removes the historical log; usually you want to *update* `thoiGianVe` and `trangThai` instead of deleting.
- **No global cascade.** Deleting the underlying `SinhVien` does *not* automatically delete their health records (Mongo has no FK cascade). Prefer changing the student's status to `Thôi học` over deletion.
- **`HoSoSucKhoe` has no indexes** (per [HoSoSucKhoe schema](../../../backend/schemas/HoSoSucKhoe.md)). Queries scan. Acceptable today; revisit if row counts climb.
