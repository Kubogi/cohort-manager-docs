# /api/tra-cuu

Two POST endpoints used by the "Tra cứu" page to find a student's grade or certificate record. Both share the same request body shape (`traCuuLookupSchema`) and require exactly one of two lookup modes.

**Roles**: `admin`, `staff`, `viewer`, `teacher` (gated at the route mount in `routes/index.js`).

**Source files**: [traCuu.route.js](../../../../backend/src/routes/traCuu.route.js), [traCuu.controller.js](../../../../backend/src/controllers/traCuu.controller.js), [traCuu.service.js](../../../../backend/src/services/traCuu.service.js), [traCuu.validator.js](../../../../backend/src/validators/traCuu.validator.js), [CertificateLookup schema](../../../backend/schemas/CertificateLookup.md).

## Endpoints

### `POST /api/tra-cuu/grades`

Look up grade records from the **primary** DB's `SinhVien` collection.

**Unit scoping applies**: staff see only students within their `allowedUnits`; teacher sees only students within `teacherScope`. Admin and viewer see all students.

**`ngaySinh` matching**: value is parsed as a date and matched as a 24-hour date range on the `ngaySinh` Date field (not a string compare).

**Response**: `{ data: GradeRow[] }` — no `meta` field.

Each `GradeRow`:
```json
{
  "stt": 1,
  "id": "<SinhVien ObjectId>",
  "maSV": "SV001",
  "hoVaTen": "Nguyễn Văn A",
  "ngaySinh": "03/04/2000",
  "lop": "K58",
  "school": { "id": "<DonViLienKet ObjectId>", "ten": "Trường CĐ Kỹ thuật", "heDaoTao": "Cao đẳng" },
  "subjectGrades": ["8.5", "7.0", ""],
  "diemTB": "7.75",
  "xepLoai": "Khá",
  "trangThai": "Đang học",
  "ghiChu": ""
}
```

**Additional errors**:
- `422 AMBIGUOUS_HE_DAO_TAO` — the student's linked school has a `heDaoTao` containing both "Đại học" and "Cao đẳng".
- `422 UNKNOWN_HE_DAO_TAO` — the student's linked school `heDaoTao` matches neither "Đại học" nor "Cao đẳng".

---

### `POST /api/tra-cuu/certificates`

Look up certificate records from the **secondary** cluster's `students` collection.

**No unit scoping** — all authenticated users with an allowed role see all records.

**`ngaySinh` matching**: string compare against `ngay_sinh` as stored in the secondary DB (format depends on source ingestion, e.g. `"03/04/2000"`).

**Response**: `{ data: CertificateRow[] }` — no `meta` field. Rows are snake_case Vietnamese — see the [CertificateLookup schema](../../../backend/schemas/CertificateLookup.md) for the field list.

---

## Shared request body

Both endpoints take the same JSON body, validated by `traCuuLookupSchema`:

```json
{
  "maSV":     "<string, optional>",
  "truong":   "<24-hex ObjectId, optional>",
  "hoTen":    "<string, optional>",
  "ngaySinh": "<dd/mm/yyyy string, optional>"
}
```

**Mode A — by name + DOB**

Provide non-empty `hoTen` and `ngaySinh`.

**Mode B — by maSV + partner school**

Provide non-empty `maSV` and `truong`.

Exactly one mode must be satisfied; otherwise the validator returns `400 VALIDATION_ERROR` with message `"Phải nhập Mã SV + Đơn vị liên kết hoặc Họ tên + Ngày sinh"`.

## Errors (both endpoints)

- `400 VALIDATION_ERROR` — neither lookup mode is satisfied, or `truong` is not a 24-hex string.
- `200 { data: [] }` — query is valid but no rows match (empty array, not a 404).
- `401 UNAUTHORIZED` / `403 FORBIDDEN` — auth (token missing/invalid/role not allowed).

## Cross-cutting notes

- **Different data sources.** `/grades` queries the primary DB (`MONGO_URI`); `/certificates` queries the secondary cluster (`MONGO_URI2`).
- **Field name style differs.** Grade rows are camelCase computed objects; certificate rows are raw snake_case Vietnamese.
- **`ngay_sinh` in the secondary DB is a string, not a Date.** Pass `"03/04/2000"`. The format depends on the source project's ingestion.
- **No write endpoints.** Both collections are read-only from this app's perspective.
- **POST, not GET.** The body avoids URL-length limits with long Vietnamese names and lets the validator return a structured 400 instead of coercing query-string strings.

## Related

- [CertificateLookup schema](../../../backend/schemas/CertificateLookup.md)
- [DonViLienKet schema](../../../backend/schemas/DonViLienKet.md) — partner-school dropdown source
- [`docs/frontend/pages/quan-ly-diem/tra-cuu.md`](../../../frontend/pages/quan-ly-diem/tra-cuu.md) — the UI
- [`docs/architecture/database.md`](../../../architecture/database.md) — two-cluster architecture
