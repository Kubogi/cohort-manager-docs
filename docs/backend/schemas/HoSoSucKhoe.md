# HoSoSucKhoe (Health Record) Schema

**Source**: [backend/src/models/HoSoSucKhoe.js](../../backend/src/models/HoSoSucKhoe.js)

**Collection**: `ho_so_suc_khoe`

**Last Verified**: 2026-05-16

---

## Overview

**CRITICAL**: This schema tracks HOSPITAL VISITS and medical leave, NOT health measurements.

It records when students go to hospitals, their diagnoses, and return times. This is a medical leave tracking system, NOT a health examination record system with vitals.

---

## Fields

| Field | Type | Required | Enum | Description |
|-------|------|----------|------|-------------|
| sinhVien | ObjectId | No | - | Student reference |
| thayQuanLy | String | No | - | Substitute manager while student is away |
| khung | String | No | 'Chính trị' \| 'Quân sự' | Training schedule being missed |
| thoiGianDi | IThoiGian | No | - | Departure time (when left for hospital) |
| thoiGianVe | IThoiGian | No | - | Return time (when returned from hospital) |
| benhVien | String | No | - | Hospital name |
| lyDo | String | No | - | Reason for visit |
| chuanDoanBenhVien | String | No | - | Hospital diagnosis |
| nguoiDiKem | String | No | 'Sinh viên' \| 'Người thân' | Who accompanied student |
| thuocDTCC | String | No | - | Special medication/treatment |
| sinhVienDiCung | String | No | - | Other students who went together |
| trangThai | String | No | 'Bình thường' \| 'Viện' | Current status |
| khoa | ObjectId | No | - | Course reference (for unit scoping) |
| daiDoi | ObjectId | No | - | Platoon reference (for unit scoping) |
| truong | ObjectId | No | - | Partner institution reference (for unit scoping) |
| createdAt | Date | Auto | - | Document creation timestamp |
| updatedAt | Date | Auto | - | Document last update timestamp |

---

## Nested Schema: IThoiGian (Time)

| Field | Type | Description |
|-------|------|-------------|
| gio | String | Time of day (HH:mm format, e.g. "14:30") |
| ngay | Date | Date |

**Note**: `thoiGianSchema` is defined with `{ _id: false }`, so sub-documents in `thoiGianDi` and `thoiGianVe` carry no `_id` field.

---

## Enums

### khung (Training Schedule)
- **Chính trị** - Political training
- **Quân sự** - Military training

### nguoiDiKem (Accompaniment)
- **Sinh viên** - Fellow student
- **Người thân** - Family member

### trangThai (Status)
- **Bình thường** - Normal/Returned
- **Viện** - Still in hospital

---

## Relationships

- **sinhVien** → SinhVien model (Many-to-One)
- **khoa** → Khoa model (Many-to-One, for unit scoping)
- **daiDoi** → DaiDoi model (Many-to-One, for unit scoping)
- **truong** → DonViLienKet model (Many-to-One, for unit scoping)

---

## Important Notes

1. **Purpose**: Tracks HOSPITAL VISITS, not health checkups/examinations
2. **All Optional**: All fields are optional in schema definition
3. **Nested Time**: thoiGianDi and thoiGianVe are objects with gio (time) and ngay (date)
4. **Status Tracking**: trangThai indicates if student is still in hospital or has returned
5. **Unit Scoping**: khoa/daiDoi/truong fields used for filtering in queries

---

## Migration from Hallucinated Documentation

**FIELDS THAT DO NOT EXIST**:
- ❌ ngayKham (examination date) - uses thoiGianDi/thoiGianVe
- ❌ chieuCao (height) - NOT a health measurements system
- ❌ canNang (weight) - NOT a health measurements system
- ❌ huyetAp (blood pressure) - NOT a health measurements system
- ❌ nhipTim (heart rate) - NOT a health measurements system
- ❌ chuanDoan (diagnosis) - actual field: chuanDoanBenhVien
- ❌ dieuTri (treatment) - not tracked
- ❌ ghiChu (notes) - not in schema

**ACTUAL PURPOSE**:
This system tracks:
- When students leave for hospital (thoiGianDi)
- Which hospital they go to (benhVien)
- Why they went (lyDo)
- What diagnosis they received (chuanDoanBenhVien)
- Who went with them (nguoiDiKem)
- When they returned (thoiGianVe)
- Their current status (still in hospital or returned)

---

## API Endpoints

**Base Path**: `/api/ho-so-suc-khoe`

- `GET /` - List health records (hospital visits)
- `GET /stats` - Get health statistics
- `GET /:id` - Get health record by ID
- `POST /` - Create new health record
- `PATCH /:id` - Update health record (**NOT PUT**)
- `DELETE /:id` - Delete health record
