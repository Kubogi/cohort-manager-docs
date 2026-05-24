# Attachment Schema

**Source**: [backend/src/models/Attachment.js](../../../backend/src/models/Attachment.js)

**Collection**: `attachments` (primary cluster)

**Last verified**: 2026-05-24 (ketQuaKhaoSat archive migration)

Polymorphic file reference. Each row points at exactly one owning record. `ownerType` is a **slot identifier** from the UI's perspective — multiple ownerTypes can share one backing mongoose model (e.g. the two survey-year slots both reference a `KhaoSatChatLuongNam` row). The `OWNER_MODEL_MAP` in `attachment.service.js` resolves ownerType → model for existence checks.

The unique `{ ownerType, ownerId }` index enforces **one attachment per (slot, owner) pair**, so two distinct slots can coexist on the same underlying owner row.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | String | Yes | — | Original filename (trimmed). Shown to users; not used for storage path. |
| `mimeType` | String | Yes | — | Set by multer. |
| `size` | Number (bytes) | Yes | — | Set by multer. |
| `storagePath` | String | Yes | — | Disk path relative to `UPLOADS_ROOT` (e.g. `attachments/QuyetDinh/<uuid>.pdf`). |
| `ownerType` | String enum | Yes | — | See `ATTACHMENT_OWNER_TYPES` in the source. Active values: `'QuyetDinh'`, `'HoSoSucKhoe'`, and the six ketQuaKhaoSat slots below. Three legacy survey values (`'KhaoSatTuDanhGiaNam'`, `'KhaoSatHoatDongTrungTamNam'`, `'KhaoSatGiangVien'`) remain in the enum so pre-migration rows stay valid; no UI writes to them. |
| `ownerId` | ObjectId | Yes | — | Reference into the model resolved by `OWNER_MODEL_MAP[ownerType]`. No `refPath` (would be incorrect for slot-style ownerTypes); use the map when populating manually. |
| `uploadedBy` | ObjectId → User | No | `null` | User who uploaded the file. |
| `createdAt`, `updatedAt` | Date | Auto | — | Mongoose timestamps. |

### ownerType → backing model

| `ownerType` | Backing model | Slot meaning |
|---|---|---|
| `QuyetDinh` | `QuyetDinh` | Scan/PDF attached to an admin decision. |
| `HoSoSucKhoe` | `HoSoSucKhoe` | Scan/PDF attached to a health record. |
| `KhaoSatGvTuDanhGiaSource` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / gvTuDanhGia** tab — uploaded source file (year-scoped). |
| `KhaoSatGvTuDanhGiaReport` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / gvTuDanhGia** tab — generated report (year-scoped). |
| `KhaoSatPhanHoiGVSource` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / phanHoiGV** tab — uploaded source file (one merged sheet per năm; not per-teacher). |
| `KhaoSatPhanHoiGVReport` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / phanHoiGV** tab — generated per-teacher report (year-scoped). |
| `KhaoSatPhanHoiTrungTamSource` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / phanHoiTrungTam** tab — uploaded source file (year-scoped). |
| `KhaoSatPhanHoiTrungTamReport` | `KhaoSatChatLuongNam` | **ketQuaKhaoSat / phanHoiTrungTam** tab — generated report (year-scoped). |
| `KhaoSatTuDanhGiaNam` | `KhaoSatChatLuongNam` | **DEPRECATED** — previously used by the giangVienTuDanhGia link page. Replaced by `KhaoSatGvTuDanhGiaSource/Report`. |
| `KhaoSatHoatDongTrungTamNam` | `KhaoSatChatLuongNam` | **DEPRECATED** — previously used by khaoSatSinhVien. Replaced by `KhaoSatPhanHoiTrungTamSource/Report`. |
| `KhaoSatGiangVien` | `GiangVienKhaoSat` | **DEPRECATED** — previously per-(teacher, năm) on khaoSatSinhVien. Replaced by the per-năm `KhaoSatPhanHoiGVSource/Report` pair (phanHoiGV is one merged sheet per năm). |

### Write authorization

`QuyetDinh` + `HoSoSucKhoe` writes (POST, DELETE) accept `admin` and `staff`.
**All nine `KhaoSat*` ownerTypes are admin-only** (the six active slots and the three deprecated ones) — non-admin POST/DELETE returns `403 FORBIDDEN`. Reads remain open per route (admin/staff/viewer/teacher).

The six active ketQuaKhaoSat slots are normally written by the
`POST /api/khao-sat-chat-luong/process` side-effect (when `nam` is provided),
not by direct `POST /api/attachments` calls. The direct route still works
for admins for parity.

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ ownerType: 1, ownerId: 1 }` | **unique** | Enforces 1 attachment per (slot, owner) pair. |

## Behavior notes

- **Disk file lifecycle.** Creating an Attachment row implies a corresponding file on disk; deleting an Attachment row must also delete the on-disk file. The service in `attachment.service.js` handles both sides. If a process crashes between the Mongo write and the disk write, you can get a row with no file (download 404s) or a file with no row (orphan). There is no consistency check.
- **Polymorphism.** `Schema.Types.ObjectId` with no `refPath` — owner model is resolved at runtime via `OWNER_MODEL_MAP[ownerType]`. `populate('ownerId')` is not safe across slot-style ownerTypes; if you need the owner, look it up explicitly with the mapped model.
- **Owner cascade.** Deleting the owning record does **not** automatically delete the Attachment — Mongo has no cascade. The owning record's delete service is responsible for clearing the attachment first.

## Related

- [`/api/attachments/*`](../../api/README.md#attachments-apiattachments) — endpoints
- [`/docs/architecture/file-storage.md`](../../architecture/file-storage.md) — on-disk layout
- [QuyetDinh](./QuyetDinh.md), [HoSoSucKhoe](./HoSoSucKhoe.md), [KhaoSatChatLuongNam](./KhaoSatChatLuongNam.md), [GiangVienKhaoSat](./GiangVienKhaoSat.md) — possible owners
