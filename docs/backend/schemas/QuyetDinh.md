# QuyetDinh (Administrative Decision) Schema

**Source**: [backend/src/models/QuyetDinh.js](../../../backend/src/models/QuyetDinh.js)

**Collection**: `quyet_dinh`

**Last verified**: 2026-05-16

---

## Overview

Administrative decisions/orders related to students (e.g., enrollment, transfer, withdrawal, disciplinary actions). Attached PDF/image files are stored in a separate [`Attachment`](Attachment.md) row (one per `(ownerType='QuyetDinh', ownerId=quyetDinh._id)`); this model does **not** carry a file path field.

---

## Fields

| Field | Type | Required | Unique | Default | Description |
|-------|------|----------|--------|---------|-------------|
| sinhVien | ObjectId → SinhVien | No | Yes* | - | Student reference (part of compound index) |
| soQD | String | No | Yes* | - | Decision number (part of compound index) |
| ngayKiQD | Date | No | No | - | Date decision was signed |
| loaiQD | String | No | No | - | Type/category of decision (free-text, no enum) |
| lyDo | String | No | No | - | Reason/justification |
| khoa | ObjectId → Khoa | No | No | - | Khoa reference (used for per-unit scope filtering) |
| daiDoi | ObjectId → DaiDoi | No | No | - | DaiDoi reference (used for per-unit scope filtering) |
| trangThai | String | No | No | - | Decision status — typically `"Chưa xử lý"` or `"Đã xử lý"`. See [decision-processing workflow](../../workflows/decision-processing.md). |
| createdAt | Date | Auto | No | - | Mongoose-managed |
| updatedAt | Date | Auto | No | - | Mongoose-managed |

\* Unique constraint is part of compound sparse index: `{ sinhVien: 1, soQD: 1 }`

---

## Typical Decision Types (loaiQD)

While not enforced by enum, typical values include:

- **Nhập học** - Enrollment
- **Chuyển đơn vị** - Transfer
- **Thôi học** - Withdrawal/Dismissal
- **Khen thưởng** - Commendation
- **Kỷ luật** - Disciplinary action
- **Tốt nghiệp** - Graduation

---

## Indexes

| Fields | Type | Unique | Sparse |
|--------|------|--------|--------|
| { sinhVien: 1, soQD: 1 } | Compound | Yes | Yes |

---

## Relationships

- **sinhVien** → SinhVien model (Many-to-One)
- **khoa** → Khoa model (Many-to-One)
- **daiDoi** → DaiDoi model (Many-to-One)

---

## Important Notes

1. **All Optional**: All fields are optional in schema definition
2. **No Enum**: loaiQD and trangThai accept any string, no enum constraint
3. **Unique Constraint**: Each student can only have one decision with a given soQD
4. **Sparse Index**: Only enforced when both sinhVien and soQD exist
5. **Attachments are separate.** PDF/image evidence lives in the [`Attachment`](Attachment.md) collection. The service layer joins them onto QuyetDinh responses via `decorateWithAttachments()`.
6. **Filtering Fields**: khoa, daiDoi, and trangThai are used for query filtering

---

## API Endpoints

**Base Path**: `/api/quyet-dinh`

- `GET /` - List decisions
- `GET /:id` - Get decision by ID
- `POST /` - Create new decision
- `PATCH /:id` - Update decision (**NOT PUT**)
- `DELETE /:id` - Delete decision

See [`docs/api/endpoints/quyet-dinh/`](../../api/endpoints/quyet-dinh/) for per-endpoint contracts and the [decision-processing workflow](../../workflows/decision-processing.md) for status-transition semantics.
