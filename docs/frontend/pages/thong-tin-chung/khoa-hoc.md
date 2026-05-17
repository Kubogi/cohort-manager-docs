# Khóa học — Cohort & course management

**Menu path**: Thông tin chung › Khóa học

**Roles**: admin · staff · viewer *(not visible to teacher)*

**Source files**: [`frontend/src/resources/thongTinChung/khoaHoc.tsx`](../../../../frontend/src/resources/thongTinChung/khoaHoc.tsx) + companion folder `khoaHoc/`

**Related API endpoints**:
- [`/api/khoa-hoc`](../../../api/endpoints/khoa/) — Khoa CRUD
- [`/api/don-vi/dai-doi`](../../../api/endpoints/don-vi/dai-doi-get-list.md) — DaiDoi CRUD
- [`/api/can-bo-quan-ly`](../../../api/endpoints/can-bo-quan-ly/) — staff
- [`/api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md) — student counts
- [`/api/truc-quan-su`](../../../api/README.md#military-duty-roster-apitruc-quan-su) — duty roster

**Related workflows**: [`auto-battalion-assignment.md`](../../../workflows/auto-battalion-assignment.md)

## When to use

Top-level cohort administration. Use it to:
- Define a new khóa (cohort) and its constituent đại đội (battalions).
- Assign management staff to battalions across time periods.
- View and edit the daily military-duty roster for that cohort.

## Layout

- Top bar: cohort selector (`SearchableSelect` of all `Khoa`).
- Tab bar: **Cán bộ quản lý** | **Trực và kiểm soát quân sự**.
- Each tab loads its own view.

## Tabs

### Tab 1 — "Cán bộ quản lý" (default)

For the selected khóa, a flat table of `cán bộ quản lý` assignments — one row per `(canBo[i], soQD[i], ngayQD[i], hieuLuc[i])` tuple — plus one placeholder row for battalions that have no staff yet (e.g. just auto-created by the Excel import on the sinh-viên page). Every battalion in the khóa appears at least once.

**What it does:**
- **Thêm khóa** — opens a modal to create a new `Khoa` with `ten` + a starting list of `DaiDoi` names. Cascading: each daiDoi is created with `khoa: <newKhoa._id>`.
- **Sửa khóa / Xoá khóa** — edits or deletes the selected khóa. Deleting cascades to its daiDois and (in some implementations) its students.
- **Thêm đại đội** — adds a new battalion to the current khóa. Validator enforces uniqueness of `(ten, khoa, donViLienKet)`.
- **Sửa đại đội** — opens a modal that includes the *aligned-arrays* editor for staff assignments: `canBo`, `soQD`, `ngayQD`, `hieuLuc` must all have equal length. The DaiDoi schema's `pre('validate')` hook enforces this at write time; the modal UI prevents you from saving misaligned arrays. The modal lists *all* current assignments for the battalion, including when there are multiple.
- **Xoá đại đội** — deletes the battalion. Cascade to students depends on service implementation.

**What it shows (table columns, left to right):**

| Column | Source |
|---|---|
| STT | row index |
| Đại đội | `DaiDoi.ten` |
| Cán bộ quản lý | `CanBoQuanLy.hoTen` (empty on placeholder rows) |
| Cấp bậc, Đơn vị, Chức vụ | from the joined `CanBoQuanLy` record |
| Số QĐ, Ngày ra QĐ, Hiệu lực | from the aligned arrays on `DaiDoi` |
| Ghi chú | from the joined `CanBoQuanLy` record |
| Hành động | edit/delete for the row's parent `DaiDoi` |

A battalion with `N` aligned staff entries produces `N` rows (same `Đại đội`, different `Cán bộ quản lý`); a battalion with none produces one placeholder row whose staff columns are blank but whose action buttons still operate on the battalion.

### Tab 2 — "Trực và kiểm soát quân sự"

Source: `TrucVaKiemSoatView.tsx`.

**What it does:**
- Calendar / list view of `TrucQuanSu` rows (daily duty roster), filtered by date range.
- Create or edit a row for a specific date: assign `trucLanhDao`, `trucChiHuy`, `truongKhungNhaD1D2`, `trucBan`, `trucQuanYNgay[0..1]`, `trucQuanYDem`, `trucDienNuoc`, `donViKiemSoatQuanSu`, `ghiChu`.

**What it shows:**
- One row per calendar date with the seven named duty roles filled in.

> The `TrucQuanSu` model has a unique index on `ngay` so each date has exactly one row. Trying to create a second row for the same date returns 409.

## Common tasks

### Create a brand-new cohort with three battalions

1. Click **Thêm khóa**, name it "Khóa 48".
2. In the same modal (or immediately after saving), add D1, D2, D3 as battalions.
3. Save. The three `DaiDoi` rows are created with `khoa: <khoa48._id>`.

### Assign a new chief to a battalion as of a given date

1. Open the battalion row, click **Sửa đại đội**.
2. In the aligned-arrays editor, append a new row: `canBo = <chief>`, `soQD = "QD-…"`, `ngayQD = …`, `hieuLuc = { batDau: <today>, ketThuc: <future> }`.
3. Save. The previous assignment's `hieuLuc.ketThuc` should be set to today's date in the same edit so the history is contiguous.

### Set the duty roster for tomorrow

1. Open **Tab 2 — Trực và kiểm soát quân sự**.
2. Pick or create the row for tomorrow.
3. Fill the seven duty roles.
4. Save.

## Edge cases / gotchas

- **The aligned-arrays editor is strict.** `DaiDoi.pre('validate')` *throws a plain `Error`* (not `HttpError`) if `canBo`, `soQD`, `ngayQD`, `hieuLuc` don't all match in length. The plain Error surfaces as a 500 with no structured code. The modal should prevent this from reaching the server, but if you see a 500 on a DaiDoi save, this is the cause.
- **Cross-khoa daiDoi mistakes.** Creating a daiDoi with `khoa: K47` and then assigning a student from `khoa K48` to it is caught at the SinhVien validator (`daiDoi.khoa` must match `payload.khoa`). The check was added in commit `7840d44`.
- **Deletion cascades are service-level, not Mongo-level.** Deleting a `Khoa` does not automatically delete its daiDois or students. The cascade in `khoa.service.remove` handles daiDois; whether it cascades to students is implementation-dependent — read the service before relying on it.
- **`trucQuanYNgay` is always length 2.** It's an array, but the Mongoose default fixes its length. Don't `push` to it.
- **No bulk operations on battalions.** Adding 20 battalions to a new cohort means 20 round-trips. Acceptable at current scale.
