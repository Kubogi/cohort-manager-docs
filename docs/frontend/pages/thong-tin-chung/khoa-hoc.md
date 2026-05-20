# Khóa học — Cohort & course management

**Menu path**: Thông tin chung › Khóa học
**Roles**: admin · staff · viewer *(not visible to teacher)*
**Source files**: [`frontend/src/resources/thongTinChung/khoaHoc.tsx`](../../../../frontend/src/resources/thongTinChung/khoaHoc.tsx) + companion folder `khoaHoc/`
**Related API endpoints**:
- [`/api/khoa-hoc`](../../../api/endpoints/khoa/) — Khoa CRUD
- [`/api/don-vi/dai-doi`](../../../api/endpoints/don-vi/dai-doi-get-list.md) — DaiDoi CRUD
- [`/api/can-bo-quan-ly`](../../../api/endpoints/can-bo-quan-ly/) — CBQL + per-khoa `phanCong[]`
- [`/api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md) — student counts
- [`/api/truc-quan-su`](../../../api/README.md#military-duty-roster-apitruc-quan-su) — duty roster

**Related workflows**: [`auto-battalion-assignment.md`](../../../workflows/auto-battalion-assignment.md)

## When to use

Top-level cohort administration. Use it to:
- Define a new khóa (cohort) and its constituent đại đội (battalions).
- Assign management staff to battalions across time periods.
- Assign a CBQL to a khóa **without** a daiDoi (orphan / khoa-only).
- View and edit the daily military-duty roster for that cohort.

## Layout

- Top bar: cohort selector (`SearchableSelect` of all `Khoa`).
- Tab bar: **Cán bộ quản lý** | **Trực và kiểm soát quân sự**.
- Each tab loads its own view.

## Tabs

### Tab 1 — "Cán bộ quản lý" (default)

For the selected khóa, a flat table built from two server queries: `GET /don-vi/dai-doi?khoa=…` (battalions) and `GET /can-bo-quan-ly?khoa=…` (every CBQL with a `phanCong` entry for that khoa). The frontend groups CBQL by `phanCong[selectedKhoa].daiDoi` and renders:

1. **Orphan rows first** — every CBQL whose `phanCong[selectedKhoa].daiDoi` is `null`. The `Đại đội` cell shows an em-dash `—`. Edit/delete on these rows opens the standalone-CBQL flow (PATCH the CBQL's phanCong entry).
2. **Battalion rows** — one row per (daiDoi, CBQL attached to that daiDoi). Battalions with no CBQL still produce one placeholder row (blank staff columns, but the action buttons still operate on the battalion).

**What it does:**
- **Thêm khóa** — opens a modal to create a new `Khoa`.
- **Sửa khóa / Xoá khóa** — edits or deletes the selected khóa. Deleting cascades to its daiDois, its students, and `$pull`s every CBQL's `phanCong` entry that pointed at this khoa.
- **Lọc theo đại đội** — a `SearchableSelect` below the khóa selector scopes the table to one battalion. Empty selection shows every battalion under the khóa (and orphans).
- **Xóa bộ lọc** — secondary button next to "Tìm kiếm"; resets the đại đội filter to empty and re-renders the full row set. Disabled when no filter is active.
- **Tìm kiếm** — leftmost action button. Re-runs the table query with the current đại đội filter.
- **Nhập excel** — uploads the `Mẫu CB QL` sheet of the K181 template and bulk-creates `CanBoQuanLy` rows. Each row's `Đại Đội` (col C) is looked up under the selected khóa and **auto-created if missing**; the `phanCong[]` entry on the CBQL stores TK/PK (col D, free text — e.g. `"Trưởng Khung Nhà D2"`), Số QĐ, Ngày ra QĐ, Hiệu lực, and Ghi chú from the same row. Rows whose `Đại Đội` value doesn't start with `c` are treated as junk and skipped entirely (no CBQL inserted). See [POST /api/can-bo-quan-ly/import](../../../api/endpoints/can-bo-quan-ly/post-import.md) for the full column mapping. The result modal lists `createdDaiDoi[]` so admins can see which battalions were synthesised.
- **Thêm đại đội** — adds a new battalion to the current khóa. Validator enforces uniqueness of `(ten, khoa, donViLienKet)`.
- **Sửa đại đội** — opens a modal that includes the staff-assignment editor. Save flow: PATCHes the DaiDoi for `{ ten }` only, then for each CBQL added/removed/changed in the lineup, PATCHes the CBQL with the updated `phanCong[]` (rewriting only the entry for the selected khoa, preserving any entries for other khoa).
- **Thêm CBQL** — new in May 2026. Opens the staff-assignment modal in standalone mode. Pick an existing CBQL, fill `số QĐ / ngày QĐ / hiệu lực / TK/PK / ghi chú`, save → the CBQL gets a `phanCong` entry with `daiDoi: null` for the selected khóa. It appears in the table as an orphan row (em-dash `Đại đội`) at the top.
- **Xóa đại đội** — deletes the battalion. CBQL with `phanCong[selectedKhoa].daiDoi === <deletedDaiDoi>` are **demoted to orphan** (`daiDoi: null`), not deleted. They keep their khoa attachment and resurface as orphan rows.
- **Xóa khóa học** — rightmost action button (danger variant). Deletes the entire khóa and all its daiDois + students after a confirmation modal showing the counts.

**What it shows (table columns, left to right):**

| Column | Source |
|---|---|
| STT | row index |
| Đại đội | `DaiDoi.ten` (or em-dash `—` for orphan rows) |
| Cán bộ quản lý | `CanBoQuanLy.hoTen` |
| Cấp bậc, Đơn vị, Chức vụ | from the `CanBoQuanLy` doc |
| Số QĐ, Ngày ra QĐ, Hiệu lực | from `CanBoQuanLy.phanCong[selectedKhoa]` |
| TK/PK | `CanBoQuanLy.phanCong[selectedKhoa].tkpk` — free text (no enum). Common values: `Trưởng khung`, `Phó khung`. Template-imported rows may include house context like `Trưởng Khung Nhà D2`. Edited via a `<input type="text">` in the StaffAssignmentModal. |
| Ghi chú | `CanBoQuanLy.phanCong[selectedKhoa].ghiChu` (per-assignment, NOT the person-level top-level `ghiChu`) |
| Hành động | edit/delete — wired to either DaiDoi handlers (battalion rows) or CBQL handlers (orphan rows) |

### Tab 2 — "Trực và kiểm soát quân sự"

Source: `TrucVaKiemSoatView.tsx`.

**Per-Khóa storage** (since 2026-05-19). Each Khóa owns its own duty calendar. The tab is empty until a Khóa is selected:

- **Khóa** — a `SearchableSelect` at the start of the filter row. Required. Until something is picked, the table stays empty, the hint "Hãy chọn khóa để xem lịch trực." is shown, and the "+ Thêm ngày" / "Nhập Excel" / "Tìm kiếm" buttons are disabled.
- **Từ ngày / Đến ngày + Tìm kiếm** — narrow the calendar to a date range within the selected Khóa.
- **Xóa lọc** — secondary button. Clears the Khóa, the date range, and the displayed entries in one action. (Pre-2026-05-19 this only reset the date range.)
- **+ Thêm ngày** — opens the entry modal; the Khóa is implicit from the page state.
- **Nhập Excel** — uploads a duty-roster workbook (sheet name containing "trực" / "truc") and stamps every row with the selected Khóa. The endpoint enforces `khoa` (`400 KHOA_REQUIRED` otherwise).

The compound `(khoa, ngay)` unique index lets two khóa share the same date.

## Common tasks

### Create a brand-new cohort with three battalions

1. Click **Thêm khóa**, name it "Khóa 48".
2. In the same modal (or immediately after saving), add D1, D2, D3 as battalions.
3. Save. The three `DaiDoi` rows are created with `khoa: <khoa48._id>`.

### Assign a CBQL to a khóa without a battalion (orphan)

1. Click **Thêm CBQL** in the action bar.
2. Pick an existing CBQL from the dropdown, fill the optional fields, save.
3. The CBQL appears in the table as an orphan row (em-dash `Đại đội`) at the top.

### Move an orphan CBQL into a battalion

1. Open the target battalion's edit modal (✏️ on its row).
2. In the staff-assignment editor, add the CBQL to the lineup.
3. Save. The CBQL's `phanCong[selectedKhoa].daiDoi` is updated from `null` to the new daiDoi id. The orphan row disappears; the battalion row gains it.

### Set the duty roster for tomorrow

1. Open **Tab 2 — Trực và kiểm soát quân sự**.
2. Pick or create the row for tomorrow.
3. Fill the seven duty roles.
4. Save.

## Edge cases / gotchas

- **Per-khoa uniqueness on `phanCong[]`.** Each CBQL may have at most one `phanCong` entry per khoa. The schema's `pre('validate')` hook surfaces `400 DUPLICATE_KHOA_IN_PHANCONG`. The PATCH path also enforces this in the service since `findOneAndUpdate` skips document middleware.
- **Cross-khoa daiDoi mismatch.** A `phanCong` entry whose `daiDoi.khoa !== entry.khoa` is rejected with `400 KHOA_DAIDOI_MISMATCH` at the service layer.
- **Deleting a Khoa pulls every phanCong entry referencing it.** This runs regardless of the cascade flag — dangling pointers to a deleted khoa would re-surface as broken rows.
- **Deleting a DaiDoi demotes phanCong entries to orphan.** The CBQL stays attached to the khoa with `daiDoi: null`; it does not get deleted.
- **Saving a battalion staff lineup fires N PATCH calls to `/can-bo-quan-ly/:id`.** One per CBQL in the diff (added, removed, or updated). At current scale (a battalion rarely has more than ~10 staff) this is acceptable. The frontend uses `Promise.all`; if any one fails the others may still land — the next refetch reflects whatever state the DB is in.
- **`trucQuanYNgay` is always length 2** (Tab 2 only). Don't `push` to it.
- **No bulk operations on battalions.** Adding 20 battalions to a new cohort means 20 round-trips.
