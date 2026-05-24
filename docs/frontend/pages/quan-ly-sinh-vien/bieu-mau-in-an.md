# Biểu mẫu & in ấn — Forms & printing

**Menu path**: Quản lý sinh viên › Biểu mẫu & in ấn
**Roles**: admin · staff · viewer · teacher *(teachers see only students within their `teacherScope`; the page has no write endpoints — Xuất Excel works for all roles)*
**Source files**: [`frontend/src/resources/quanLyGiangDay/bieuMauInAn/bieuMauInAn.tsx`](../../../../frontend/src/resources/quanLyGiangDay/bieuMauInAn/bieuMauInAn.tsx) + companion folder

> The entry file is parked under `quanLyGiangDay/` for historical reasons even though the menu surfaces it under "Quản lý sinh viên". Don't move it without updating the import in `App.tsx`.

**Related API endpoints**: [`GET /api/bieu-mau/export`](../../../api/README.md#excel-form-export-apibieu-mau)
**Related workflows**: [`excel-export-forms.md`](../../../workflows/excel-export-forms.md), [`/docs/architecture/excel-pipeline.md`](../../../architecture/excel-pipeline.md)

## When to use

Generate a printable Excel grade-book or attendance roster filled with current student data. Pick the **hệ** (Đại học or Cao đẳng), the **type of list** (`danhSach`), and (for grade lists) the **subject** (`mon`). Optionally narrow by `khoa`, `đại đội`, `truong`. Click **Xuất Excel** to download.

## Layout

- Filter bar: `khoa`, `đại đội`, `truong`, `hệ`, `danh sách`, `môn`.
- Conditional logic:
  - When `hệ = Đại học`, the `danh sách` dropdown shows the 5 Đại học templates.
  - When `hệ = Cao đẳng`, the dropdown shows the 6 Cao đẳng templates.
  - When the chosen template is `multiByMon` type, the `môn` dropdown becomes optional; leaving it blank fills every subject sheet.
- **Xuất Excel** button at the bottom.
- A summary panel shows the current filter state for sanity-check before exporting.

## Common tasks

### Print the attendance roster for K47/D1

1. `hệ = Đại học`, `danh sách = Danh sách điểm danh`.
2. `khoa = K47`, `đại đội = D1`.
3. **Xuất Excel**. Browser downloads `DS điểm danh.xlsx` populated.

### Print the final-grade list for "Quân sự chung"

1. `hệ = Đại học`, `danh sách = Danh sách điểm hết học phần`.
2. `môn = Quân sự chung`.
3. Optionally narrow by `khoa`. Export.

### Print the full grade book

1. `hệ = Đại học`, `danh sách = Sổ điểm từng môn`.
2. Pick `khoa` (and optionally `daiDoi` for one battalion's grade book).
3. Export. The resulting workbook has one block per subject per student.

## Edge cases / gotchas

- **`Danh sách điểm miệng` exists only for Cao đẳng.** Đại học students don't have an oral component.
- **`multiByMon` templates write one sheet per subject.** If you leave `môn` blank, every subject sheet is populated. To get only one subject, pick `môn` explicitly.
- **Validation gaps.** The route has no Joi schema (per [`backend.md`](../../../architecture/backend.md)); the controller enforces requirements by hand. Errors come back as `400 INVALID_HE`, `INVALID_TEMPLATE`, `INVALID_MON`, or `MISSING_PARAMS`. The UI hides most invalid combinations, but unusual filter states (no khoa selected, single khoa with no students) produce empty workbooks rather than errors.
- **Template files are committed to git.** If you need to add a column or change formatting, edit the `.xlsx` in `backend/forms/Hệ {ĐH,CĐ}/` and commit it. Don't generate templates dynamically.
- **Per-unit scoping applies.** A staff user only sees students in their `allowedUnits` reflected in the exported workbook.
