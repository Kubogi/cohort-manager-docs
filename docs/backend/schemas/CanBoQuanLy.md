# CanBoQuanLy (Management Staff) Schema

**Source**: [backend/src/models/CanBoQuanLy.js](../../../backend/src/models/CanBoQuanLy.js)
**Collection**: `can_bo_quan_ly`
**Last verified**: 2026-05-19

---

## Overview

Management staff (cán bộ quản lý) — instructors, platoon commanders, and other personnel who oversee students. Optionally linked to a `User` account so the staff member can log in as a `teacher`. Per-khoa/per-daiDoi assignments are stored on this model in the `phanCong[]` array.

---

## Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `hoTen` | String | **Yes** | — | Full name (trimmed) |
| `capBac` | String | No | `''` | Military rank (e.g. "Thiếu tá", "Đại tá") |
| `donViQL` | String | No | `''` | Managing unit / workplace assignment |
| `chucVu` | String | No | `''` | Position / role title |
| `soDienThoai` | String | No | `''` | Phone number |
| `ghiChu` | String | No | `''` | Person-level free-text notes (distinct from `phanCong[i].ghiChu`, which is per-assignment) |
| `phanCong` | PhanCong[] | No | `[]` | Per-khoa assignments. See sub-schema below. |
| `taiKhoan` | ObjectId → `User` | No | `null` | Optional link to a `User` account; allows the staff member to log in (typically as `teacher`) |
| `createdAt` | Date | Auto | — | Mongoose-managed |
| `updatedAt` | Date | Auto | — | Mongoose-managed |

`hoTen` is the only required field. Every string field defaults to `''` (not `undefined`) so that read patterns can rely on string operations without null-checks.

---

## Nested Schema: PhanCong (per-khoa assignment)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `khoa` | ObjectId → `Khoa` | **Yes** | — | Which khoa this assignment applies to |
| `daiDoi` | ObjectId → `DaiDoi` | No | `null` | Optional battalion within the khoa; `null` means "khoa-only / orphan" — the CBQL surfaces as an em-dash row in the khoaHoc CBQL tab |
| `soQD` | String | No | `''` | Decision number for this assignment |
| `ngayRaQD` | Date | No | `null` | Date the decision was issued |
| `hieuLuc` | `{ batDau: Date, ketThuc: Date }` | No | `{}` | Validity period |
| `tkpk` | String | No | `''` | Free text (max 200 chars). Common values are `'Trưởng khung'` / `'Phó khung'`, but the K181 import template ships richer strings like `'Trưởng Khung Nhà D2'` that include house context — stored verbatim. |
| `ghiChu` | String | No | `''` | Per-assignment free-text note (max 500 chars). Distinct from the top-level `ghiChu`. |

---

## Invariants on `phanCong[]`

1. **At most one entry per `khoa`** per CBQL — enforced by a `pre('validate')` hook (sync, no DB lookup). API responses surface this as `400 DUPLICATE_KHOA_IN_PHANCONG`.
2. **If `daiDoi` is set, `DaiDoi.khoa` must equal the entry's `khoa`** — enforced at the service layer in [canBoQuanLy.service.js](../../../backend/src/services/canBoQuanLy.service.js) (`validateCrossKhoaDaiDoi`) since it needs a DB lookup. API responses surface this as `400 KHOA_DAIDOI_MISMATCH`.
3. **Deleting a `Khoa` pulls every `phanCong` entry referencing it** — [khoa.service.js](../../../backend/src/services/khoa.service.js) runs `updateMany($pull)` regardless of the `cascade` flag.
4. **Deleting a `DaiDoi` demotes matching entries to orphan** (`daiDoi: null`) rather than pulling them — [donVi.service.js](../../../backend/src/services/donVi.service.js) `removeDaiDoi`.

---

## Indexes

| Fields | Type | Unique | Notes |
|--------|------|--------|-------|
| `{ hoTen: 1 }` | Single | No | Speeds up the menu's typeahead by-name lookup |
| `{ 'phanCong.khoa': 1 }` | Single | No | Speeds up `?khoa=` list filter |
| `{ 'phanCong.daiDoi': 1 }` | Single | No | Speeds up "which daiDoi a CBQL is attached to" lookups |
| `{ taiKhoan: 1 }` | Single | **Yes** | Partial — only enforced when `taiKhoan` is a real `ObjectId`; rows with `taiKhoan: null` are excluded via `partialFilterExpression: { taiKhoan: { $type: 'objectId' } }`. |

---

## Relationships

- **`taiKhoan` → `User`** — one-to-one (optional). If set, this staff member can log in as that user.
- **`phanCong[].khoa` → `Khoa`** — many-to-many across multiple khoa.
- **`phanCong[].daiDoi` → `DaiDoi`** — optional per-khoa attachment to a battalion.

`phanCong` is the **source of truth** for CBQL ↔ Khoa/DaiDoi assignments. The old aligned-arrays pattern on `DaiDoi` (`canBo[]/soQD[]/ngayQD[]/hieuLuc[]/tkpk[]/ghiChu[]`) was removed in May 2026.

---

## Important notes

1. **`capBac` is the rank, `chucVu` is the position** — both fields exist; do not collapse them.
2. **`taiKhoan` is an ObjectId**, not a username string. To find a staff row by their login username, look the username up in `User` first, then find the `CanBoQuanLy` whose `taiKhoan` equals that `User._id`.
3. **The partial index on `taiKhoan`** means seeding many rows with `taiKhoan: null` is safe; the unique constraint only kicks in when a real ObjectId is written.
4. **Top-level `ghiChu` vs `phanCong[i].ghiChu`** — the former is person-level (set by Excel import, shared across all assignments); the latter is per-khoa-assignment. The khoaHoc CBQL table column "Ghi chú" reads `phanCong[selectedKhoa].ghiChu`.

---

## API endpoints

**Base Path**: `/api/can-bo-quan-ly` — `teacher` role is excluded at the route mount in `routes/index.js`.

- `GET /` — list staff. Query: `hoTen`, `donVi`, `chucVu`, `khoa` (filters to CBQL with a phanCong entry for that khoa), `unattached=true` (combined with `khoa`, returns only CBQL whose phanCong entry has `daiDoi: null`).
- `GET /:id` — fetch one
- `POST /` — create (admin)
- `PATCH /:id` — update (admin)
- `DELETE /:id` — remove (admin)
- `POST /import` — Excel bulk import (admin) — requires the `khoa` form field. Each imported CBQL gets a `phanCong` entry pointing at the matched (or auto-created) DaiDoi. See [post-import.md](../../api/endpoints/can-bo-quan-ly/post-import.md) for the K181 template column mapping.

See [`docs/api/endpoints/can-bo-quan-ly/`](../../api/endpoints/can-bo-quan-ly/) for per-endpoint contracts.

---

## Migration

Old data lived as aligned arrays on `DaiDoi`. A one-shot migration script at [`backend/scripts/migrate-cbql-phancong.js`](../../../backend/scripts/migrate-cbql-phancong.js) lifts every `DaiDoi.canBo[i] / soQD[i] / …` slot into a `CanBoQuanLy.phanCong` entry, then unsets the legacy fields. Idempotent — re-running is safe.
