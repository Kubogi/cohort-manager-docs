# CanBoQuanLy (Management Staff) Schema

**Source**: [backend/src/models/CanBoQuanLy.js](../../../backend/src/models/CanBoQuanLy.js)

**Collection**: `can_bo_quan_ly`

**Last verified**: 2026-05-16

---

## Overview

Management staff (cán bộ quản lý) — instructors, platoon commanders, and other personnel who oversee students. Optionally linked to a `User` account so the staff member can log in as a `teacher`.

---

## Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `hoTen` | String | **Yes** | — | Full name (trimmed) |
| `capBac` | String | No | `''` | Military rank (e.g. "Thiếu tá", "Đại tá") |
| `donViQL` | String | No | `''` | Managing unit / workplace assignment |
| `chucVu` | String | No | `''` | Position / role title |
| `soDienThoai` | String | No | `''` | Phone number |
| `ghiChu` | String | No | `''` | Free-text notes |
| `soQD` | String | No | `''` | Reference number of the decision that assigned this staff member |
| `ngayRaQD` | String | No | `''` | Date the decision was issued (free-text, not a `Date` type) |
| `hieuLuc` | String | No | `''` | Effective-status text (e.g. "Còn hiệu lực", "Hết hiệu lực") |
| `taiKhoan` | ObjectId → `User` | No | `null` | Optional link to a `User` account; allows the staff member to log in (typically as `teacher`) |
| `createdAt` | Date | Auto | — | Mongoose-managed |
| `updatedAt` | Date | Auto | — | Mongoose-managed |

`hoTen` is the only required field. Every string field defaults to `''` (not `undefined`) so that read patterns can rely on string operations without null-checks.

---

## Indexes

| Fields | Type | Unique | Notes |
|--------|------|--------|-------|
| `{ hoTen: 1 }` | Single | No | Speeds up the menu's typeahead by-name lookup |
| `{ taiKhoan: 1 }` | Single | **Yes** | Partial — only enforced when `taiKhoan` is a real `ObjectId`; rows with `taiKhoan: null` are excluded via `partialFilterExpression: { taiKhoan: { $type: 'objectId' } }`. This lets many rows have no linked account while still preventing two staff rows from pointing at the same `User`. |

---

## Relationships

- **`taiKhoan` → `User`** — one-to-one (optional). If set, this staff member can log in as that user.
- **Inverse: `DaiDoi.canBo[]`** — `DaiDoi` arrays reference `CanBoQuanLy._id` to assign management to a battalion. Aligned-array invariant: see [DaiDoi schema](DaiDoi.md).
- `khoa` and `daiDoi` are **not** fields on this model — staff are linked to units only via the inverse `DaiDoi.canBo` reference.

---

## Important notes

1. **`capBac` is the rank, `chucVu` is the position** — both fields exist; do not collapse them.
2. **`taiKhoan` is an ObjectId**, not a username string. To find a staff row by their login username, look the username up in `User` first, then find the `CanBoQuanLy` whose `taiKhoan` equals that `User._id`.
3. **`soQD` / `ngayRaQD` / `hieuLuc` are free-text strings** for the original appointment decision — not foreign keys to a `QuyetDinh` row.
4. **The partial index on `taiKhoan`** means seeding many rows with `taiKhoan: null` is safe; the unique constraint only kicks in when a real ObjectId is written.

---

## API endpoints

**Base Path**: `/api/can-bo-quan-ly` — `teacher` role is excluded at the route mount in `routes/index.js`.

- `GET /` — list staff
- `GET /:id` — fetch one
- `POST /` — create (admin)
- `PATCH /:id` — update (admin)
- `DELETE /:id` — remove (admin)
- `POST /import` — Excel bulk import (admin)

See [`docs/api/endpoints/can-bo-quan-ly/`](../../api/endpoints/can-bo-quan-ly/) for per-endpoint contracts.
