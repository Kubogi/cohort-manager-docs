# Excel Export — Forms & Grade Books

**Triggered from**: [Biểu mẫu & in ấn](../frontend/pages/quan-ly-sinh-vien/bieu-mau-in-an.md) page; **Xuất Excel** button on grade-related pages.

**Touches**: `GET /api/bieu-mau/export`, `backend/forms/Hệ {ĐH,CĐ}/*.xlsx` templates, `exceljs`.

**Who can do this**: `admin`, `staff`, `viewer`, `teacher`.

## Goal

Render an `.xlsx` printable form pre-filled with current student / grade data. The user picks a template via `(hệ, danhSách, môn)` and optional unit filters; the backend opens the matching template file from `backend/forms/`, fills the cells, and streams the binary back.

## Happy path

1. User selects `hệ` (Đại học / Cao đẳng). The `danh sách` dropdown updates to show valid templates for that hệ.
2. User picks `danh sách`. `mon` is **always required** for every template — the service guard is `if (!he || !danhSach || !mon) throw 400 MISSING_PARAMS`, unconditionally.
3. User optionally narrows by `khoa`, `daiDoi`, `truong`. Defaults to "all in scope".
4. Click **Xuất Excel** → `GET /api/bieu-mau/export?he=…&danhSach=…&mon=…&khoa=…&daiDoi=…&truong=…`.
5. Backend:
   - Looks up the template in `TEMPLATE_MAP[he][danhSach]`. 400 on mismatch.
   - Loads the file from `backend/forms/<file>` via `exceljs`.
   - Queries `SinhVien` with the resolved filters + `applyUnitScope` / `applyTeacherScope`.
   - Writes rows into the right cells per the template's `type` and `sheetMode`. See [`docs/architecture/excel-pipeline.md`](../architecture/excel-pipeline.md).
   - Streams `workbook.xlsx.write(res)` back as a binary attachment.
6. Browser downloads. User opens it in Excel for review or printing.

## Side-effects

- **None on the DB.** Pure read.
- **Templates are untouched** — `backend/forms/` is read-only by convention.

## Failure modes

| Scenario | Result |
|---|---|
| `(he, danhSach)` not in `TEMPLATE_MAP` | `400 INVALID_TEMPLATE` |
| `mon` not in `MON_SHEET_INDEX[he]` for a `multiByMon` template | `400 INVALID_MON` |
| Required param missing | `400 MISSING_PARAMS` |
| Template file gone from disk (deployment issue) | `500 TEMPLATE_ERROR` |
| Filters resolve to zero students | Empty workbook returned. No error. |

## Manual test recipe

- [ ] Export `(Đại học, "Danh sách điểm danh")` for K47/D1. Confirm the resulting `.xlsx` opens in Excel with student rows populated and an empty attendance column.
- [ ] Export `(Đại học, "Sổ điểm từng môn")` for K47 without `daiDoi`. Confirm every daiDoi's students appear.
- [ ] Export `(Cao đẳng, "Danh sách điểm miệng", "Chính trị 1")` for K47. Confirm `mieng` values are populated.
- [ ] Try `(Đại học, "Danh sách điểm miệng")` — there's no such template for Đại học. Expect `400 INVALID_TEMPLATE`.
- [ ] As a teacher restricted to D1, export a Sổ điểm. Confirm only D1 students appear regardless of the filter.
