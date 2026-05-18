# SinhVien (Student) Schema

**Source**: [backend/src/models/sinhVien.js](../../../backend/src/models/sinhVien.js)
**Collection**: `sinh_vien`
**Last verified**: 2026-05-18

---

## Overview

Student records in the system. Each student belongs to a class (lop), platoon (daiDoi), course (khoa), and partner institution (truong). Grades are embedded as subdocuments within each student document.

---

## Fields

| Field | Type | Required | Unique | Constraints | Description |
|-------|------|----------|--------|-------------|-------------|
| maSV | String | No | Yes* | - | Student ID (part of compound index) |
| cccd | String | No | No | - | Citizen-ID number |
| hoTen | String | No | Yes* | - | Full name (part of compound index) |
| ngaySinh | Date | No | Yes* | - | Date of birth (part of compound index) |
| nganh | String | No | No | - | Major/specialization/field of study |
| noiSinh | String | No | No | - | Place of birth |
| danToc | String | No | No | - | Ethnicity (e.g. "Kinh", "Tày") |
| gioiTinh | **Boolean** | No | No | - | Gender (true = Male, false = Female) |
| soDienThoai | String | No | No | - | Phone number |
| lop | String | No | No | - | Class designation/name |
| daiDoi | ObjectId | No | No | ref: 'DaiDoi' | Platoon reference |
| khoa | ObjectId | No | No | ref: 'Khoa' | Course reference |
| truong | ObjectId | No | No | ref: 'DonViLienKet' | Partner institution (secondary DB) |
| ngayNhapHoc | Date | No | No | - | Enrollment date |
| ngayVe | Date | No | No | - | Return/departure date (if applicable) |
| trangThai | String | No | No | enum: 'Đang học' / 'Hoãn học' / 'Thôi học' / 'Đình chỉ' / 'Học ghép' / 'Miễn học' / 'Không tham gia học' | Current academic status. Mutations are typically driven by the [decision-processing workflow](../../workflows/decision-processing.md). `'Học ghép'` (joined-class) students have per-subject `tbMon` computed normally, but the **cross-subject `diemTB` summary is suppressed** (rendered as `—` in `TongKetCuoiKhoaView` and returned as `''` by the xếp loại tally — they land in `chuaCoDiem`). |
| ghiChu | String | No | No | - | General notes/comments |
| ghiChuYTe | String | No | No | default: `''` | Free-text medical note shown alongside `trangThaiSucKhoe` in the health-record UI. |
| trangThaiSucKhoe | String | No | No | - | Health status summary |
| monMienHoc | [String] | No | No | enum: MON_ENUM, default: `[]` | Subjects the student is exempt from. **Writes to `SinhVien.diem` for an exempt subject are rejected** by the diem service with `409 SUBJECT_EXEMPT` (covers `POST /api/diem`, `PATCH /api/diem/:id`, and `POST /api/diem/import` — the importer skips exempt rows with a soft error). The Nhập điểm UI renders dashes in place of input cells for exempt rows. Setting/clearing the field is part of the student add/edit modal via the "Môn miễn học" sub-popup. |
| diem | DiemSubdocument[] | No | No | Embedded array | Grades (NOT separate collection) |
| khoaSortKey | Number | No | No | default: `Number.MAX_SAFE_INTEGER`, indexed | **Denormalized sort key.** `Khoa.ten` parsed as a number (`"K47"` → `47`, `"169"` → `169`). Rows whose khoa is unknown or has no digits sort last. Automatically resolved on save/update of `khoa`; cascaded by `Khoa.post('save')` and `Khoa.post('findOneAndUpdate')` when a Khoa is renamed. |
| daiDoiSortKey | Number | No | No | default: `Number.MAX_SAFE_INTEGER`, indexed | **Denormalized sort key.** `DaiDoi.ten` with leading `c`/`C` (and any other non-digit prefix) stripped, then parsed as a number (`"c11"` → `11`). Same hook chain as `khoaSortKey`. |
| createdAt | Date | Auto | No | - | Document creation timestamp |
| updatedAt | Date | Auto | No | - | Document last update timestamp |

\* Unique constraint is part of compound sparse index: `{ maSV: 1, hoTen: 1, ngaySinh: 1 }`

---

## Embedded Subdocument: DiemSubdocument

**This is NOT a separate collection** - grades are embedded within each student document.

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| _id | ObjectId | Auto | - | Subdocument ID |
| mon | String | Yes | Enum: MON_ENUM (7 courses) | Course name |
| thuongXuyen | Number | Yes | 0-10 | Regular/continuous assessment score |
| mieng | Number | No | 0-10 | Oral exam score (only for certain courses) |
| giuaHP | Number | Yes | 0-10 | Midterm exam score |
| hetHP | Number | Yes | 0-10 | Final exam score |
| tbMon | Number | Auto | 0-10 | Average score (auto-calculated) |
| createdAt | Date | Auto | - | Subdocument creation timestamp |
| updatedAt | Date | Auto | - | Subdocument last update timestamp |

### MON_ENUM Values

1. Đường lối quốc phòng và an ninh của ĐCSVN
2. Công tác Quốc phòng và An ninh
3. Quân sự chung
4. Kỹ thuật chiến đấu bộ binh và chiến thuật
5. Chính trị 1
6. Chính trị 2
7. Quân sự

### Grade Calculation Formula

**For first 4 courses**:
```
tbMon = thuongXuyen * 0.1 + giuaHP * 0.3 + hetHP * 0.6
```

**For last 3 courses**:
```
tbMon = thuongXuyen * 0.1 + mieng * 0.1 + giuaHP * 0.2 + hetHP * 0.6
```

---

## Indexes

| Fields | Type | Unique | Sparse |
|--------|------|--------|--------|
| { maSV: 1, hoTen: 1, ngaySinh: 1 } | Compound | Yes | Yes |
| { khoaSortKey: 1, daiDoiSortKey: 1, _id: 1 } | Compound | No | No |
| { khoaSortKey: 1 } | Single | No | No |
| { daiDoiSortKey: 1 } | Single | No | No |

The compound sort-key index backs the canonical student ordering applied by every list endpoint and Excel export (see [pagination convention](../../frontend/conventions/pagination.md)). Within a `(khoa, daiDoi)` group, `_id` provides insertion-order ordering. The single-field indexes support filter queries that pin a khoa or đại đội before sorting within.

---

## Relationships

- **daiDoi** → DaiDoi model (Many-to-One)
- **khoa** → Khoa model (Many-to-One)
- **truong** → DonViLienKet model (Many-to-One) - Uses secondary database

---

## Special Considerations

1. **Grades Storage**: Grades are embedded, not a separate collection. To query grades, use aggregation pipeline or access via parent student document.
2. **Gender Field**: Boolean type (true = Male, false = Female) - **NOT** string enum ['Nam', 'Nữ']
3. **Database**: Uses default Mongoose connection (MONGO_URI)
4. **Compound Unique Index**: Sparse index on (maSV, hoTen, ngaySinh) - only enforced when all three fields exist
5. **Auto-calculation**: `tbMon` in grades is auto-calculated by pre-validate hook - do not set manually
6. **No Required Fields**: All fields are optional in schema definition
7. **Denormalized sort keys**: `khoaSortKey` and `daiDoiSortKey` are derived from the referenced `Khoa.ten` / `DaiDoi.ten`. They are kept in sync by Mongoose hooks (`pre('save')`, `pre('insertMany')`, `pre('findOneAndUpdate'/'updateOne'/'updateMany')`) plus cascading post-hooks on Khoa/DaiDoi when those documents are renamed. The within-`(khoa, daiDoi)` order is plain `_id` ascending — insertion order, since Mongo generates sequential ObjectIds for `insertMany` and any later `create()`. Existing data is backfilled by [`backend/src/scripts/backfill-student-sort-keys.js`](../../../backend/src/scripts/backfill-student-sort-keys.js). Helpers live in [`backend/src/utils/sortKeys.js`](../../../backend/src/utils/sortKeys.js).

---

## Accessing Grades

Since grades are embedded, use one of these approaches:

**Option 1**: Access via parent document
```javascript
const student = await SinhVien.findById(id);
const grades = student.diem;
```

**Option 2**: Use aggregation to "flatten" grades
```javascript
await SinhVien.aggregate([
  { $unwind: '$diem' },
  { $match: { 'diem.mon': 'Chính trị 1' } }
]);
```

---

## API Endpoints

**Base Path**: `/api/sinh-vien`

- `GET /` - List students (with unit scoping)
- `GET /:id` - Get student by ID
- `POST /` - Create new student
- `PATCH /:id` - Update student (**NOT PUT**)
- `DELETE /:id` - Delete student
- `POST /import` - Import students from file
- `GET /export/all` - Export all students
