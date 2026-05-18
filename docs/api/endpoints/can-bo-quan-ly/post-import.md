# POST /api/can-bo-quan-ly/import

**Endpoint**: `POST /api/can-bo-quan-ly/import`
**Authentication**: ✅ Required
**Roles**: admin
**Content-Type**: `multipart/form-data`
**Last Verified**: 2026-05-17

---

## Description

Bulk-imports management staff (`CanBoQuanLy`) from an Excel file. When the request includes a `khoa` form field, each newly-inserted CBQL is **also attached** to the matching `DaiDoi` under that khóa via the aligned-arrays `canBo`/`soQD`/`ngayQD`/`hieuLuc` slots (with placeholder QĐ values that an admin can fill in later).

This endpoint is admin-only.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

### Form fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | Yes | Excel file (`.xls` / `.xlsx`), max 20 MB |
| `khoa` | string (ObjectId) | No | If present, scopes the daiDoi-linkage step to that khóa. Without it, CBQL records are inserted but not linked to any đại đội. |

---

## Excel format

### Sheet matching

The importer picks the first worksheet whose name, lowercased and stripped of whitespace, contains `cbql`. Examples that match: `"Mẫu CB QL"`, `"  CBQL Khóa 175 "`, `"DS CBQL"`. Examples that don't: `"Roster"`, `"Trực kiểm soát"`. Workbooks with no matching sheet return `400 NO_CBQL_SHEET`.

### Row structure

The header sits on **row 1**; data starts at **row 2**. A row whose column A (`TT`) is non-empty marks the start of a new person; subsequent rows with empty `TT` are treated as continuation rows of the previous person. Each person therefore spans 1–2 rows in practice.

| Col | Field | Behavior |
|-----|-------|----------|
| A | `TT` (序) | Boundary marker — non-empty = new person |
| B | `hoTen` | Required; comes from the first row of the person. Empty → row ignored. |
| C | `capBac` | Stored verbatim (e.g. `"4//"`, `"3//"`). |
| D | `chucVu` | Multi-line — the first row's value is the primary position; continuation rows are joined with `"; "`. |
| E | `ghiChu` | Free text (e.g. school name). The first non-empty value across the person's rows is kept. |
| F | đại đội code | E.g. `"c14"`, `"c18"`. Lowercased before lookup. Used together with `khoa` to find the `DaiDoi` (`DaiDoi.findOne({ ten, khoa })`). |

### Dedup

By `hoTen` only. A name already in the DB → counted under `duplicates`, no re-insert. Even duplicates can still be attached to a daiDoi in the same import call.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "inserted": 9,
    "duplicates": 2,
    "attached": 11,
    "missingDaiDoi": ["c99"]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `inserted` | number | New CBQL rows created. |
| `duplicates` | number | Rows skipped because `hoTen` already exists in `CanBoQuanLy`. |
| `attached` | number | Rows pushed into a DaiDoi's `canBo` array (with aligned placeholder soQD/ngayQD/hieuLuc). Includes both new and previously-existing CBQL whose name matched a row. Zero when `khoa` is omitted. |
| `missingDaiDoi` | string[] | Distinct daiDoi codes from column F that didn't match any `DaiDoi` under the given `khoa`. |

---

## Error responses

| Status | Code | Cause |
|--------|------|-------|
| 400 | `EMPTY_BODY` (no code) | No file in the request. |
| 400 | `NO_CBQL_SHEET` | No worksheet matched `cbql`. |
| 403 | (auth) | Caller is not admin. |

---

## Notes

- The temp upload file is always cleaned up in a `finally` block, even when the service throws.
- The DaiDoi schema enforces aligned arrays via a `pre('validate')` hook. The importer pushes `''` / `null` / `{}` to keep `canBo`/`soQD`/`ngayQD`/`hieuLuc` the same length when adding a new staff entry.
- The previous version of this endpoint expected sheet `"DS CBQL"` rows 4+ with a fill-color-based `donViQL` heuristic. That contract is **removed** — see the commit history if you have old exports relying on it.

---

## Related

- [POST /api/can-bo-quan-ly](./post.md)
- [GET /api/can-bo-quan-ly](./get-list.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
- [Khóa học page](../../../frontend/pages/thong-tin-chung/khoa-hoc.md) — the UI that calls this endpoint
