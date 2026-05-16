# /api/attachments

Polymorphic file attachments for `QuyetDinh` (administrative decisions) and `HoSoSucKhoe` (health records). One attachment per owning record, enforced by the unique `{ ownerType, ownerId }` index on the `Attachment` collection.

Roles: per-route gating lives inside the route file (the mount in `routes/index.js` has no outer `authorize()`). Effective per-method roles:

| Method | Roles |
|---|---|
| `GET /` (by owner) | `admin`, `staff`, `viewer`, `teacher` |
| `POST /` (upload) | `admin`, `staff` |
| `GET /:id/download` | `admin`, `staff`, `viewer`, `teacher` |
| `DELETE /:id` | `admin`, `staff` |

**Source files**: [attachment.route.js](../../../../backend/src/routes/attachment.route.js), [attachment.controller.js](../../../../backend/src/controllers/attachment.controller.js), [attachment.service.js](../../../../backend/src/services/attachment.service.js), [Attachment schema](../../../backend/schemas/Attachment.md).

## Endpoints

### `GET /api/attachments?ownerType=&ownerId=`

Look up the single attachment metadata row for a given owning record. Used by the QĐ and Hồ sơ sức khỏe row components to render the "attached file" column.

**Query**: `ownerType` (`"QuyetDinh"` or `"HoSoSucKhoe"`), `ownerId` (24-hex ObjectId).

**Response**: `{ data: AttachmentRow | null }`. `null` when no attachment exists for the owner.

### `POST /api/attachments`

Upload + create. `multipart/form-data`.

**Fields**: `file` (required, ≤ 100 MB per `MAX_FILE_SIZE` in [`storage.js`](../../../../backend/src/config/storage.js); multer fileFilter checks `ALLOWED_MIMETYPES`), `ownerType` (`"QuyetDinh"` or `"HoSoSucKhoe"`), `ownerId` (ObjectId of the owning record).

**Response 201**: `{ data: { _id, name, mimeType, size, storagePath, ownerType, ownerId, uploadedBy } }`.

**Errors**:
- `400 INVALID_FILE` — missing or unsupported file
- `404 OWNER_NOT_FOUND` — the referenced QuyetDinh or HoSoSucKhoe doesn't exist
- `409 ATTACHMENT_EXISTS` — the owning record already has an attachment (unique index hit). Delete first.

### `GET /api/attachments/:id/download`

Stream the binary.

**Response**: binary stream with `Content-Disposition: attachment; filename="<name>"` and the original `mimeType`. **Not** a JSON envelope.

**Errors**: `404 NOT_FOUND` (row gone OR on-disk file missing).

### `DELETE /api/attachments/:id`

Delete both the Mongo row AND the on-disk file. **No rollback** if the disk delete fails after the Mongo delete — possible orphan file remains.

**Response**: `204 No Content`.

**Errors**: `404 NOT_FOUND`.

## Cross-cutting notes

- **One attachment per owning record** — strict invariant from the unique index. Workflows that need multiple files per record (e.g. several scans of one decision) must change the schema first.
- **Owner cascade is application-level, not Mongo-level.** Deleting the owning `QuyetDinh` or `HoSoSucKhoe` does **not** automatically delete the Attachment. The owning entity's delete service is responsible for clearing the attachment first. Verify this before relying on it.
- **Polymorphic via `refPath: 'ownerType'`** — use `populate('ownerId')` to follow the reference.
- **Per-unit scoping doesn't apply directly here** — but the OWNING record's scope does. A staff user who can't see a `QuyetDinh` shouldn't have its attachment ID, so the downstream attachment routes are effectively scoped through the owner.

## Related

- [Attachment schema](../../../backend/schemas/Attachment.md)
- [`docs/architecture/file-storage.md`](../../../architecture/file-storage.md)
- [QuyetDinh schema](../../../backend/schemas/QuyetDinh.md), [HoSoSucKhoe schema](../../../backend/schemas/HoSoSucKhoe.md) — the two valid owners
