# DaiDoi (Platoon) Schema

**Source**: [backend/src/models/DaiDoi.js](../../backend/src/models/DaiDoi.js)
**Collection**: `dai_doi`
**Last Verified**: 2026-05-19

---

## Overview

Represents a DaiDoi (platoon/battalion) within a Khoa (course).

CBQL ↔ DaiDoi assignments **no longer live on this model** — as of May 2026 they have moved to `CanBoQuanLy.phanCong[].daiDoi`. See [CanBoQuanLy](CanBoQuanLy.md) for the full assignment shape (soQD, ngayRaQD, hieuLuc, tkpk, ghiChu).

---

## Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| ten | String | Yes | Platoon name (e.g. "Đại đội 1", "DD2") |
| khoa | ObjectId | No | Course reference |
| donViLienKet | ObjectId | **Yes** | Partner institution reference. Every DaiDoi has exactly one DonViLienKet — see invariant below. |
| ngayBatDau | Date | No | Platoon start date |
| ngayKetThuc | Date | No | Platoon end date |
| quanSo | Number | No | Platoon strength/headcount |
| createdAt | Date | Auto | Document creation timestamp |
| updatedAt | Date | Auto | Document last update timestamp |

---

## Indexes

| Fields | Type | Unique |
|--------|------|--------|
| { ten: 1, khoa: 1, donViLienKet: 1 } | Compound | Yes |

---

## Relationships

- **khoa** → Khoa model (Many-to-One)
- **donViLienKet** → DonViLienKet model (Many-to-One)
- **Inverse: `CanBoQuanLy.phanCong[].daiDoi`** — staff assignments are read from this inverse reference, not stored on DaiDoi itself.

---

## Organizational Hierarchy

```
Truong (Partner Institution)
  └── Khoa (Course/Department)
      └── DaiDoi (Platoon) <- THIS LEVEL
          └── SinhVien (Students)
```

---

## Deletion semantics

Deleting a DaiDoi does **not** delete its associated CBQL. Instead, `CanBoQuanLy.phanCong` entries that referenced this `daiDoi` have their `daiDoi` field set to `null` — the CBQL stays attached to the khoa and surfaces as an orphan row (em-dash Đại đội) in the khoaHoc CBQL tab.

---

## Important Notes

1. **Required Fields**: `ten` and `donViLienKet` are required. Every DaiDoi must be attached to exactly one DonViLienKet — see the invariant below.
2. **Compound Unique Index**: (ten, khoa, donViLienKet) ensures unique platoon names within context.
3. **Used as Organizational Level**: Between Khoa and SinhVien.
4. **Rename cascade**: Renaming `ten` triggers a post-save/post-findOneAndUpdate hook that recomputes `SinhVien.daiDoiSortKey` for every student in this đại đội. The parser strips a leading `c`/`C` then takes the leading digit run (`"c11"` → `11`), so renaming `c11` → `C11` is a no-op for ordering. See [SinhVien](SinhVien.md) and [sortKeys.js](../../../backend/src/utils/sortKeys.js).

---

## Invariant: every DaiDoi has exactly one DonViLienKet

The `donViLienKet` reference is required at both the schema layer (`required: true`) and the create validator (`daiDoiCreateSchema` in [donVi.validator.js](../../../backend/src/validators/donVi.validator.js)). The Excel student importer ([sinhVien.service.js](../../../backend/src/services/sinhVien.service.js)) refuses to run without a `truong` body field and passes it into every auto-created DaiDoi.

This invariant is what makes `(khoa, truong) → DaiDoi[]` queries reliable — both the **Tổng kết cuối khóa** export and the partner-school dropdown rely on it.

Existing data is backfilled by [`backend/src/scripts/backfill-daidoi-donvi.js`](../../../backend/src/scripts/backfill-daidoi-donvi.js).

---

## API Endpoints

**Base Path**: `/api/don-vi/dai-doi`

- `GET /dai-doi` — List all platoons. Query: `khoa`, `khoaId`, `canBo` (resolves to `DaiDoi`s referenced by that CBQL's `phanCong[].daiDoi`).
- `GET /dai-doi/:id` — Get platoon by ID
- `POST /dai-doi` — Create new platoon
- `PATCH /dai-doi/:id` — Update platoon (**NOT PUT**)
- `DELETE /dai-doi/:id` — Delete platoon (demotes CBQL phanCong entries to orphan)
