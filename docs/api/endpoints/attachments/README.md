# /api/attachments

Polymorphic file attachments. `ownerType` is a **slot identifier**: most map 1-to-1 to a mongoose model, but two of the survey slots share a single backing model (see [Attachment schema](../../../backend/schemas/Attachment.md#ownertype--backing-model)). One attachment per `(ownerType, ownerId)` pair, enforced by the unique index on the `Attachment` collection.

Roles: per-route gating lives inside the route file (the mount in `routes/index.js` has no outer `authorize()`). The route-level role check is permissive; the service-layer check additionally restricts the three `KhaoSat*` ownerTypes to admin on writes:

| Method | Route-level roles | Additional service-layer restriction |
|---|---|---|
| `GET /` (by owner) | `admin`, `staff`, `viewer`, `teacher` | — |
| `POST /` (upload) | `admin`, `staff` | `KhaoSat*` ownerTypes → admin only |
| `GET /:id/download` | `admin`, `staff`, `viewer`, `teacher` | — |
| `DELETE /:id` | `admin`, `staff` | `KhaoSat*` ownerTypes → admin only |

**Source files**: [attachment.route.js](../../../../backend/src/routes/attachment.route.js), [attachment.controller.js](../../../../backend/src/controllers/attachment.controller.js), [attachment.service.js](../../../../backend/src/services/attachment.service.js), [Attachment schema](../../../backend/schemas/Attachment.md).

## Endpoints

### `GET /api/attachments?ownerType=&ownerId=`

Look up the single attachment metadata row for a given owning record. Used by the QĐ and Hồ sơ sức khỏe row components to render the "attached file" column.

**Query**: `ownerType` (one of `'QuyetDinh'`, `'HoSoSucKhoe'`, `'KhaoSatTuDanhGiaNam'`, `'KhaoSatHoatDongTrungTamNam'`, `'KhaoSatGiangVien'`), `ownerId` (24-hex ObjectId).

**Response**: `{ data: AttachmentRow | null }`. `null` when no attachment exists for the owner.

### `POST /api/attachments`

Upload + create. `multipart/form-data`.

**Fields**: `file` (required, ≤ 100 MB per `MAX_FILE_SIZE` in [`storage.js`](../../../../backend/src/config/storage.js); multer fileFilter checks `ALLOWED_MIMETYPES`), `ownerType` (one of the five values listed above), `ownerId` (ObjectId of the owning record).

If a row already exists for `(ownerType, ownerId)`, this endpoint **upserts** — the new file replaces the old one and the prior on-disk file is unlinked. No 409 in normal use; the unique-index conflict path is only reachable under a race.

**Response 201**: `{ data: { _id, name, mimeType, size, storagePath, ownerType, ownerId, uploadedBy } }`.

**Errors**:
- `400 BAD_OWNER_TYPE` — `ownerType` not in the allowed enum
- `400 BAD_OWNER_ID` — `ownerId` missing / not a valid ObjectId
- `400 INVALID_FILE_TYPE` — extension not in `ALLOWED_EXTENSIONS`
- `403 FORBIDDEN` — caller's role can't write to this `ownerType` (e.g. staff uploading a `KhaoSat*` file)
- `404 OWNER_NOT_FOUND` — the referenced owner row doesn't exist

### `GET /api/attachments/:id/download`

Stream the binary.

**Response**: binary stream with `Content-Disposition: attachment; filename="<name>"` and the original `mimeType`. **Not** a JSON envelope.

**Errors**: `404 NOT_FOUND` (row gone OR on-disk file missing).

### `DELETE /api/attachments/:id`

Delete both the Mongo row AND the on-disk file. **No rollback** if the disk delete fails after the Mongo delete — possible orphan file remains.

**Response**: `204 No Content`.

**Errors**: `403 FORBIDDEN` (non-admin on a `KhaoSat*` ownerType), `404 NOT_FOUND`.

## Cross-cutting notes

- **One attachment per `(ownerType, ownerId)` pair** — strict invariant from the unique index. Two distinct ownerTypes pointing at the same underlying row (e.g. both KhaoSat year-slots on one `KhaoSatChatLuongNam`) coexist fine.
- **Owner cascade is application-level, not Mongo-level.** Deleting the owning record does **not** automatically delete the Attachment. The owning entity's delete service is responsible for clearing the attachment first. Verify this before relying on it.
- **Polymorphism via `OWNER_MODEL_MAP`** — no `refPath`; `populate('ownerId')` is not safe across slot-style ownerTypes. Look the owner up explicitly via the mapped model when needed.
- **Per-unit scoping doesn't apply directly here** — but the OWNING record's scope does. A staff user who can't see a `QuyetDinh` shouldn't have its attachment ID, so the downstream attachment routes are effectively scoped through the owner.

## Related

- [Attachment schema](../../../backend/schemas/Attachment.md)
- [`docs/architecture/file-storage.md`](../../../architecture/file-storage.md)
- [QuyetDinh schema](../../../backend/schemas/QuyetDinh.md), [HoSoSucKhoe schema](../../../backend/schemas/HoSoSucKhoe.md), [KhaoSatChatLuongNam schema](../../../backend/schemas/KhaoSatChatLuongNam.md), [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md) — possible owners
