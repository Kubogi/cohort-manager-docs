# /api/truc-quan-su

Daily military-duty roster, **scoped per Khóa**. One row per `(khoa, ngay)` pair.

**Roles**: `admin`, `staff`, `viewer`. **`teacher` is excluded** (mounted with `authorize(['admin', 'staff', 'viewer'])` in `routes/index.js`).

**Source files**: [trucQuanSu.route.js](../../../../backend/src/routes/trucQuanSu.route.js), [trucQuanSu.controller.js](../../../../backend/src/controllers/trucQuanSu.controller.js), [trucQuanSu.service.js](../../../../backend/src/services/trucQuanSu.service.js), [TrucQuanSu schema](../../../backend/schemas/trucQuanSu.md).

## Endpoints

### `GET /api/truc-quan-su`

List roster rows for a Khóa. **Ordered by `ngay` ascending** (oldest date first).

**Query**: `khoa` (ObjectId — **required**; without it the response is `{ data: [], meta: { total: 0, ... } }`), `from` and `to` (ISO dates, optional date range), `page`, `limit` (default: 200).

**Response**: `{ data: TrucQuanSuItem[], meta: { total, page, limit } }`. Item shape:
```
{
  _id, khoa, ngay,
  trucLanhDao, trucChiHuy, truongKhungNhaD1D2,
  trucBan, trucQuanYNgay: [string, string], trucQuanYDem,
  trucDienNuoc, donViKiemSoatQuanSu, ghiChu,
  createdAt, updatedAt
}
```

### `GET /api/truc-quan-su/:id`

One row by ID.

**Response**: `{ data: TrucQuanSuItem }`. `404 NOT_FOUND` if absent.

### `POST /api/truc-quan-su`

**Roles**: `admin`, `staff` only. Viewer cannot create rows.

Create a row for a `(khoa, date)` pair.

**Body**: At minimum `{ khoa: <ObjectId>, ngay: <ISO date> }`. All other fields default to empty strings (or `["", ""]` for `trucQuanYNgay`). `khoa` is required by Joi. Unique on `(khoa, ngay)` — a second row for the same khoa + date returns `409 DUPLICATE_NGAY`; two different khóa can share the same date.

**Response 201**: `{ data: TrucQuanSuItem }`.

### `PATCH /api/truc-quan-su/:id`

**Roles**: `admin`, `staff` only. Viewer cannot update rows.

Update any of the duty-role fields for that row.

**Body**: Any subset of the role fields, including `ngay` and `khoa` (both can be changed). If the new `(khoa, ngay)` collides with an existing row, returns `409 DUPLICATE_NGAY`.

**Response**: `{ data: TrucQuanSuItem }`.

### `DELETE /api/truc-quan-su/:id`

**Roles**: `admin`, `staff` only. Viewer cannot delete rows.

Remove a row.

**Response**: `204 No Content`.

### `POST /api/truc-quan-su/import`

**Roles**: `admin`, `staff` only.
**Content-Type**: `multipart/form-data`.

Bulk-import the duty roster from an Excel file into a chosen Khóa. The importer picks the first worksheet whose name, lowercased and stripped of whitespace, contains `trực` or `truc` (e.g. `"trực kiểm soát QS"`). No matching sheet → `400 NO_TRUC_SHEET`. Missing `khoa` form field → `400 KHOA_REQUIRED`.

**Sheet layout**: header on rows 1–4 (merged), data starts at **row 5**, **2 rows per entry**:

| Col | Field | Row 1 of pair | Row 2 of pair |
|---|---|---|---|
| A | day-of-week / date | `"Thứ Hai"` | `"12/29/25"` (M/D/YY); also accepted as a real Date cell |
| B–E, G–J | name fields | name | concatenated with row 1 via a space (handles names that wrap across the two rows) |
| F | `trucQuanYNgay[0]` / `[1]` | first quân-y day-shift person | **separate** second quân-y day-shift person |

`ngay` is taken from row 2 col A. Pairs with an unparseable date are skipped silently (this lets the importer tolerate trailing blank rows).

**Form fields**:

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | File | Yes | `.xls` / `.xlsx`, max 20 MB |
| `khoa` | string (ObjectId) | **Yes** | The Khóa under which to insert the imported rows. |

**Response 200**:

```json
{
  "data": {
    "inserted": 42,
    "duplicates": 0,
    "errors": []
  }
}
```

Duplicate `(khoa, ngay)` (already in DB or repeated in the file) is counted under `duplicates` rather than failing the request. The schema's compound unique index on `{khoa, ngay}` is what backs this guarantee.

## Cross-cutting notes

- **Per-Khóa storage.** Schedules belong to a specific Khóa. `GET` without `khoa` is an empty result, not an error. The frontend tab gates fetches on a Khóa SearchableSelect.
- **`trucQuanYNgay` is always length 2.** Don't push more entries; replace the array verbatim.
- **`ngay` is a Mongo `Date`** — clients should send ISO strings (`"2026-05-16"`). Mongoose coerces.
- **Per-unit scoping (`allowedUnits`) does not apply.** The roster is unit-agnostic within a Khóa.
- **No partial-day rosters.** One row covers a whole calendar day; there's no concept of multi-shift / multi-row per date.
- **Migration:** the pre-2026-05-19 schema was a single global calendar (unique on `ngay`). The rollout dropped all existing rows via [`backend/scripts/drop-truc-quan-su.js`](../../../../backend/scripts/drop-truc-quan-su.js) (chosen over backfill — the Excel import recreates the calendar trivially).

## Related

- [TrucQuanSu schema](../../../backend/schemas/trucQuanSu.md)
- [`docs/frontend/pages/thong-tin-chung/khoa-hoc.md`](../../../frontend/pages/thong-tin-chung/khoa-hoc.md) — UI on Tab 2 "Trực và kiểm soát quân sự"
