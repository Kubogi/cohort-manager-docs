# DaiDoi (Platoon) Schema

**Source**: [backend/src/models/DaiDoi.js](../../backend/src/models/DaiDoi.js)  
**Collection**: `dai_doi`  
**Last Verified**: 2026-05-18

---

## Overview

Represents a DaiDoi (platoon) within a Khoa (course).

**IMPORTANT**: Contains complex parallel array structure (canBo, soQD, ngayQD, hieuLuc, tkpk, ghiChu) that must maintain equal lengths.

---

## Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| ten | String | Yes | Platoon name (e.g. "Đại đội 1", "DD2") |
| khoa | ObjectId | No | Course reference |
| donViLienKet | ObjectId | **Yes** | Partner institution reference. Every DaiDoi has exactly one DonViLienKet — see invariant below. |
| ngayBatDau | Date | No | Platoon start date |
| ngayKetThuc | Date | No | Platoon end date |
| canBo | ObjectId[] | No | Array of staff references (aligned) |
| soQD | String[] | No | Array of decision numbers (aligned) |
| ngayQD | Date[] | No | Array of decision dates (aligned) |
| hieuLuc | IHieuLuc[] | No | Array of validity periods (aligned) |
| tkpk | String[] | No | Array of TK/PK roles per assignment (aligned). Enum: `''`, `'Trưởng khung'`, `'Phó khung'`. |
| ghiChu | String[] | No | Array of per-assignment free-text notes (aligned, max 500 chars each). Distinct from the person-level `CanBoQuanLy.ghiChu` field — that one is set by the Excel import and shared across all assignments. |
| quanSo | Number | No | Platoon strength/headcount |
| createdAt | Date | Auto | Document creation timestamp |
| updatedAt | Date | Auto | Document last update timestamp |

---

## Nested Schema: IHieuLuc (Validity Period)

| Field | Type | Description |
|-------|------|-------------|
| batDau | Date | Start date of validity period |
| ketThuc | Date | End date of validity period |

**Note**: `hieuLucSchema` is defined with `{ _id: false }`, so sub-documents in `hieuLuc[]` carry no `_id` field.

---

## Parallel Array Structure

**CRITICAL**: Six arrays (canBo, soQD, ngayQD, hieuLuc, tkpk, ghiChu) must maintain equal lengths.

Each index represents one management assignment:
- canBo[i] = staff member
- soQD[i] = decision number for that assignment
- ngayQD[i] = date of that decision
- hieuLuc[i] = validity period for that assignment
- tkpk[i] = TK/PK role for that assignment (`''` | `'Trưởng khung'` | `'Phó khung'`)
- ghiChu[i] = per-assignment note for that assignment

### Example

```javascript
{
  canBo: [ObjectId("staff1"), ObjectId("staff2")],
  soQD: ["QĐ-001/2024", "QĐ-002/2024"],
  ngayQD: [Date("2024-01-15"), Date("2024-06-01")],
  hieuLuc: [
    { batDau: Date("2024-01-15"), ketThuc: Date("2024-05-31") },
    { batDau: Date("2024-06-01"), ketThuc: Date("2024-12-31") }
  ],
  tkpk: ["Trưởng khung", "Phó khung"],
  ghiChu: ["Phụ trách khung D2", ""]
}
```

This represents two management assignments:
- staff1 assigned by QĐ-001/2024 on 2024-01-15 as Trưởng khung with the note "Phụ trách khung D2", valid until 2024-05-31
- staff2 assigned by QĐ-002/2024 on 2024-06-01 as Phó khung, valid until 2024-12-31

---

## Indexes

| Fields | Type | Unique |
|--------|------|--------|
| { ten: 1, khoa: 1, donViLienKet: 1 } | Compound | Yes |

---

## Validation

Pre-validate hook checks the six parallel arrays (canBo, soQD, ngayQD, hieuLuc, tkpk, ghiChu). The validator passes if ALL six arrays are empty, OR if every non-empty array shares the same length. A mix of empty and non-empty arrays passes as long as all non-empty ones agree on length — this is what lets legacy DaiDoi docs predating `tkpk`/`ghiChu` continue to validate on resave.

Throws error: "canBo, soQD, ngayQD, hieuLuc, tkpk, and ghiChu must have the same length"

---

## Relationships

- **khoa** → Khoa model (Many-to-One)
- **donViLienKet** → DonViLienKet model (Many-to-One)
- **canBo** → CanBoQuanLy model (Many-to-Many via array)

---

## Organizational Hierarchy

```
Truong (Partner Institution)
  └── Khoa (Course/Department)
      └── DaiDoi (Platoon) <- THIS LEVEL
          └── SinhVien (Students)
```

---

## Important Notes

1. **Required Fields**: `ten` and `donViLienKet` are required. Every DaiDoi must be attached to exactly one DonViLienKet — see the invariant below.
2. **Parallel Arrays**: Six arrays must maintain equal lengths (validated)
3. **Compound Unique Index**: (ten, khoa, donViLienKet) ensures unique platoon names within context
4. **Used as Organizational Level**: Between Khoa and SinhVien
5. **Rename cascade**: Renaming `ten` triggers a post-save/post-findOneAndUpdate hook that recomputes `SinhVien.daiDoiSortKey` for every student in this đại đội. The parser strips a leading `c`/`C` then takes the leading digit run (`"c11"` → `11`), so renaming `c11` → `C11` is a no-op for ordering. See [SinhVien](SinhVien.md) and [sortKeys.js](../../../backend/src/utils/sortKeys.js).

---

## Invariant: every DaiDoi has exactly one DonViLienKet

The `donViLienKet` reference is required at both the schema layer (`required: true`) and the create validator (`daiDoiCreateSchema` in [donVi.validator.js](../../../backend/src/validators/donVi.validator.js)). The Excel student importer ([sinhVien.service.js](../../../backend/src/services/sinhVien.service.js)) refuses to run without a `truong` body field and passes it into every auto-created DaiDoi.

This invariant is what makes `(khoa, truong) → DaiDoi[]` queries reliable — both the **Tổng kết cuối khóa** export and the partner-school dropdown rely on it. Without the invariant, an auto-created DaiDoi could exist with `donViLienKet: null` and silently disappear from those queries.

Existing data is backfilled by [`backend/src/scripts/backfill-daidoi-donvi.js`](../../../backend/src/scripts/backfill-daidoi-donvi.js), which derives the value from the students referencing each DaiDoi. Run it once on each environment before deploying the `required: true` flip.

---

## Migration from Hallucinated Documentation

**FIELDS THAT DO NOT EXIST**:
- ❌ maDaiDoi (platoon code/ID) - Does not exist, only has ten (name)
- ❌ moTa (description) - Not in schema
- ❌ tenDaiDoi (platoon name) - Actual field is just "ten", not "tenDaiDoi"

**COMPLEX STRUCTURES NOT DOCUMENTED**:
- ✅ Parallel arrays (canBo, soQD, ngayQD, hieuLuc, tkpk, ghiChu) with validation
- ✅ donViLienKet reference
- ✅ quanSo field

---

## API Endpoints

**Base Path**: `/api/don-vi/dai-doi`

- `GET /dai-doi` - List all platoons
- `GET /dai-doi/:id` - Get platoon by ID
- `POST /dai-doi` - Create new platoon
- `PATCH /dai-doi/:id` - Update platoon (**NOT PUT**)
- `DELETE /dai-doi/:id` - Delete platoon
