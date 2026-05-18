# DaiDoi (Platoon) Schema

**Source**: [backend/src/models/DaiDoi.js](../../backend/src/models/DaiDoi.js)  
**Collection**: `dai_doi`  
**Last Verified**: 2026-05-16

---

## Overview

Represents a DaiDoi (platoon) within a Khoa (course).

**IMPORTANT**: Contains complex parallel array structure (canBo, soQD, ngayQD, hieuLuc) that must maintain equal lengths.

---

## Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| ten | String | Yes | Platoon name (e.g. "Đại đội 1", "DD2") |
| khoa | ObjectId | No | Course reference |
| donViLienKet | ObjectId | No | Partner institution reference |
| ngayBatDau | Date | No | Platoon start date |
| ngayKetThuc | Date | No | Platoon end date |
| canBo | ObjectId[] | No | Array of staff references (aligned) |
| soQD | String[] | No | Array of decision numbers (aligned) |
| ngayQD | Date[] | No | Array of decision dates (aligned) |
| hieuLuc | IHieuLuc[] | No | Array of validity periods (aligned) |
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

**CRITICAL**: Four arrays (canBo, soQD, ngayQD, hieuLuc) must maintain equal lengths.

Each index represents one management assignment:
- canBo[i] = staff member
- soQD[i] = decision number for that assignment
- ngayQD[i] = date of that decision
- hieuLuc[i] = validity period for that assignment

### Example

```javascript
{
  canBo: [ObjectId("staff1"), ObjectId("staff2")],
  soQD: ["QĐ-001/2024", "QĐ-002/2024"],
  ngayQD: [Date("2024-01-15"), Date("2024-06-01")],
  hieuLuc: [
    { batDau: Date("2024-01-15"), ketThuc: Date("2024-05-31") },
    { batDau: Date("2024-06-01"), ketThuc: Date("2024-12-31") }
  ]
}
```

This represents two management assignments:
- staff1 assigned by QĐ-001/2024 on 2024-01-15, valid until 2024-05-31
- staff2 assigned by QĐ-002/2024 on 2024-06-01, valid until 2024-12-31

---

## Indexes

| Fields | Type | Unique |
|--------|------|--------|
| { ten: 1, khoa: 1, donViLienKet: 1 } | Compound | Yes |

---

## Validation

Pre-validate hook checks the four parallel arrays (canBo, soQD, ngayQD, hieuLuc). The validator passes if ALL four arrays are empty, OR if every non-empty array shares the same length. A mix of empty and non-empty arrays passes as long as all non-empty ones agree on length.

Throws error: "canBo, soQD, ngayQD, and hieuLuc must have the same length"

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

1. **Only Required Field**: `ten` is the only required field
2. **Parallel Arrays**: Four arrays must maintain equal lengths (validated)
3. **Compound Unique Index**: (ten, khoa, donViLienKet) ensures unique platoon names within context
4. **Used as Organizational Level**: Between Khoa and SinhVien
5. **Rename cascade**: Renaming `ten` triggers a post-save/post-findOneAndUpdate hook that recomputes `SinhVien.daiDoiSortKey` for every student in this đại đội. The parser strips a leading `c`/`C` then takes the leading digit run (`"c11"` → `11`), so renaming `c11` → `C11` is a no-op for ordering. See [SinhVien](SinhVien.md) and [sortKeys.js](../../../backend/src/utils/sortKeys.js).

---

## Migration from Hallucinated Documentation

**FIELDS THAT DO NOT EXIST**:
- ❌ maDaiDoi (platoon code/ID) - Does not exist, only has ten (name)
- ❌ moTa (description) - Not in schema
- ❌ tenDaiDoi (platoon name) - Actual field is just "ten", not "tenDaiDoi"

**COMPLEX STRUCTURES NOT DOCUMENTED**:
- ✅ Parallel arrays (canBo, soQD, ngayQD, hieuLuc) with validation
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
