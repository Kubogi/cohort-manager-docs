# GET /api/bieu-mau/export

**Endpoint**: `GET /api/bieu-mau/export`

**Authentication**: ✅ Required (Bearer)

**Roles**: `admin`, `staff`, `viewer`, `teacher`

**Source**: [bieuMau.route.js](../../../../backend/src/routes/bieuMau.route.js), [bieuMau.controller.js](../../../../backend/src/controllers/bieuMau.controller.js), [bieuMau.service.js](../../../../backend/src/services/bieuMau.service.js), [config/bieuMau.config.js](../../../../backend/src/config/bieuMau.config.js)

**Last verified**: 2026-05-16

## Description

Generate a populated Excel grade-book or roster `.xlsx`. The backend picks a template based on `(he, danhSach)`, optionally filters students by `(khoa, daiDoi, truong)`, writes per-subject grades into the right cells, and streams the resulting binary back to the client.

The full template-map and per-row column layout is in [`docs/architecture/excel-pipeline.md`](../../../architecture/excel-pipeline.md).

## Request

```
GET /api/bieu-mau/export
  ?he=Đại học | Cao đẳng                  (required)
  &danhSach=<one of the templates>          (required; values vary per he)
  &mon=<one of MON_ENUM>                    (required for singleGrade/soDiem; optional for multiByMon defaults)
  &khoa=<ObjectId>                          (optional filter)
  &daiDoi=<ObjectId>                        (optional filter)
  &truong=<ObjectId>                        (optional filter)
  &status=<comma-separated trạng thái>      (optional; e.g. "Đang học,Hoãn học")
```

`status` accepts a single enum value or a comma-separated list; multi-value is matched via `$in` against `SinhVien.trangThai`. Omitting the param applies no status filter.

The route has **no Joi validator**; the service enforces the contract by hand.

### Valid `danhSach` values per `he`

**Đại học:**
- `Danh sách điểm danh`
- `Danh sách điểm thường xuyên`
- `Danh sách điểm kiểm tra giữa học phần`
- `Danh sách điểm hết học phần`
- `Sổ điểm từng môn`

**Cao đẳng:**
- `Danh sách điểm danh`
- `Danh sách điểm miệng`
- `Danh sách điểm thường xuyên`
- `Danh sách điểm kiểm tra giữa học phần`
- `Danh sách điểm hết học phần`
- `Sổ điểm các môn`

### Valid `mon` values per `he`

**Đại học:** the first 4 of `MON_ENUM` — `Đường lối quốc phòng và an ninh của ĐCSVN`, `Công tác Quốc phòng và An ninh`, `Quân sự chung`, `Kỹ thuật chiến đấu bộ binh và chiến thuật`.

**Cao đẳng:** the remaining 3 — `Chính trị 1`, `Chính trị 2`, `Quân sự`.

## Response

### 200 OK — binary stream

```
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="<sanitized name>.xlsx"
```

Body is the workbook bytes. **Not** a JSON envelope.

### Errors

| Status | code | When |
|---|---|---|
| 400 | `MISSING_PARAMS` | `he` or `danhSach` not in the query |
| 400 | `INVALID_HE` | `he` not in `{"Đại học", "Cao đẳng"}` |
| 400 | `INVALID_TEMPLATE` | `(he, danhSach)` not in `TEMPLATE_MAP` |
| 400 | `INVALID_MON` | `mon` not valid for the chosen `he` |
| 401 | `UNAUTHORIZED` | Missing/expired/bad access token |
| 403 | `FORBIDDEN` | Role check failed |
| 500 | `TEMPLATE_ERROR` | Template file missing on disk (deployment problem) |

## Examples

```bash
# Print the final-exam list for K47/D1 in Quân sự chung
curl -o "DS điểm hết HP.xlsx" \
  -H "Authorization: Bearer $TOKEN" \
  "https://your-domain/api/bieu-mau/export?he=Đại+học&danhSach=Danh+sách+điểm+hết+học+phần&mon=Quân+sự+chung&khoa=K47&daiDoi=D1"

# Full grade book for K48
curl -o "Sổ điểm.xlsx" \
  -H "Authorization: Bearer $TOKEN" \
  "https://your-domain/api/bieu-mau/export?he=Đại+học&danhSach=Sổ+điểm+từng+môn&khoa=K48"
```

## Behavior notes

- **`multiByMon` template + no `mon`** = every subject sheet in the template is populated. Useful for full grade books.
- **`multiByMon` template + specific `mon`** = only that subject's sheet is populated; others are left as the template defines them.
- **Per-unit scoping applies** — `applyUnitScope` (staff) or `applyTeacherScope` (teacher) filters which students appear. Admin sees all.
- **`status` is applied before the workbook fill**, so the resulting roster only contains students whose `trangThai` is in the supplied set. Useful for printing rosters of e.g. only `Đang học` students.
- **`tbMon` is read from the SinhVien.diem subdocument as-is** — the exporter does **not** recompute, so the integer-arithmetic guarantees from the schema hook are preserved.
- **Empty filter results produce an empty workbook**, not an error. If your students don't appear, check the filters.

## Related

- [`docs/architecture/excel-pipeline.md`](../../../architecture/excel-pipeline.md) — template map + column layout
- [`backend/forms/`](../../../../backend/forms/) — the `.xlsx` template files themselves
- [SinhVien schema](../../../backend/schemas/SinhVien.md) — embedded `diem[]` shape
- [Glossary: hệ, danh sách, môn](../../../glossary.md)
