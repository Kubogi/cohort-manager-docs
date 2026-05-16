# CertificateLookup Schema

**Source**: [backend/src/models/CertificateLookup.js](../../../backend/src/models/CertificateLookup.js)

**Collection**: `students` **(secondary cluster — `MONGO_URI2`)**

**Last verified**: 2026-05-16

Mirror of the `students` collection in the [Certificate Management](https://github.com/Kubogi/certificate-management-docs) system. Read-only from this app's perspective — the lookup project owns writes; this app just reads to support the public "Tra cứu" page (look up a certificate by student name + DOB, or by maSV + sending school).

Field names are **snake_case Vietnamese** because that's the on-disk schema of the source-of-truth project. Don't try to align it with `SinhVien` (which is camelCase Vietnamese) — different cluster, different owner.

> ⚠ This collection's name (`students`) clashes visually with `sinh_vien` in the primary cluster. They are different. `sinh_vien` is mutable, owned by this app; `students` is the snake-case certificate archive populated by the external project.

## Fields

| Field | Type | Description |
|---|---|---|
| `truong_id` | ObjectId | Reference to the partner school in the secondary cluster's `schools` collection (i.e. a [DonViLienKet](./DonViLienKet.md)). |
| `ma_sinh_vien` | String | Student ID code (mirrors `SinhVien.maSV`). |
| `cccd` | String | Citizen ID. |
| `ho_va_ten` | String | Full name. |
| `ngay_sinh` | String | Date of birth (stored as a string in the source project; not a Date). |
| `noi_sinh` | String | Place of birth. |
| `lop` | String | Class. |
| `xep_loai` | String | Final classification (e.g. "Giỏi", "Khá"). |
| `so_hieu_chung_chi` | String | Certificate serial number. |
| `so_vao_so` | String | Certificate ledger number. |
| `khoa` | String | Cohort label (free text; **not** an ObjectId to `Khoa`). |
| `nam` | String | Year of issue. |
| `so_quyet_dinh` | String | Source decision number. |

No `createdAt` / `updatedAt` — `timestamps: false`.

## Indexes

None defined in this app's schema. The source-of-truth project may have its own indexes on the same physical collection; this app reads with whatever is there. Queries that scan a large `students` collection may be slow until an index is added externally.

## Behavior notes

- **Read-only.** This app never writes to `students`. The lookup project does. If you need to back-fill or sync, do it through that project's interfaces.
- **Mongoose alias is NOT used here.** Unlike `DonViLienKet` (which aliases `ten ↔ ten_truong`), this schema exposes the raw snake_case names directly. Code that reads `CertificateLookup` documents sees Vietnamese snake_case fields.
- **String dates.** `ngay_sinh` is a string, format determined by the source project (commonly `dd/mm/yyyy`). Don't try to compare it as a `Date` without parsing first.
- **Used by**: the `/api/tra-cuu` endpoint (`backend/src/services/traCuu.service.js`).

## Related

- [DonViLienKet](./DonViLienKet.md) — the partner-school side, also in the secondary cluster
- [`/api/tra-cuu`](../../api/README.md#public-lookup-apitra-cuu) — endpoint that queries this collection
- [`/docs/architecture/database.md`](../../architecture/database.md) — two-connection architecture
