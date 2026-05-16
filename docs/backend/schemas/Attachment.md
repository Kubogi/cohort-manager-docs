# Attachment Schema

**Source**: [backend/src/models/Attachment.js](../../../backend/src/models/Attachment.js)

**Collection**: `attachments` (primary cluster)

**Last verified**: 2026-05-16

Polymorphic file reference. Each row points at exactly one owning record — either a `QuyetDinh` (administrative decision) or a `HoSoSucKhoe` (health record). Used by the UI to attach a scanned PDF / image / Word doc to a workflow item.

The unique `{ ownerType, ownerId }` index enforces **one attachment per record**. To swap the file, delete first then re-upload, or the second upload will 409.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | String | Yes | — | Original filename (trimmed). Shown to users; not used for storage path. |
| `mimeType` | String | Yes | — | Set by multer. |
| `size` | Number (bytes) | Yes | — | Set by multer. |
| `storagePath` | String | Yes | — | Disk path relative to `UPLOADS_ROOT` (e.g. `attachments/QuyetDinh/<uuid>.pdf`). |
| `ownerType` | String enum | Yes | — | One of `'QuyetDinh'`, `'HoSoSucKhoe'` (constant `ATTACHMENT_OWNER_TYPES` in the source). |
| `ownerId` | ObjectId | Yes | — | Polymorphic reference; `refPath: 'ownerType'` so `populate()` works. |
| `uploadedBy` | ObjectId → User | No | `null` | User who uploaded the file. |
| `createdAt`, `updatedAt` | Date | Auto | — | Mongoose timestamps. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ ownerType: 1, ownerId: 1 }` | **unique** | Enforces 1 attachment per owning record. |

## Behavior notes

- **Disk file lifecycle.** Creating an Attachment row implies a corresponding file on disk; deleting an Attachment row must also delete the on-disk file. The service in `attachment.service.js` handles both sides. If a process crashes between the Mongo write and the disk write, you can get a row with no file (download 404s) or a file with no row (orphan). There is no consistency check.
- **Polymorphism via `refPath`.** `Schema.Types.ObjectId` + `refPath: 'ownerType'` lets `populate('ownerId')` follow the reference to the right collection. Use this in services when surfacing the owner alongside the attachment.
- **Owner cascade.** Deleting the owning `QuyetDinh` or `HoSoSucKhoe` does **not** automatically delete the Attachment — Mongo has no cascade. The owning record's delete service is responsible for clearing the attachment first.

## Related

- [`/api/attachments/*`](../../api/README.md#attachments-apiattachments) — endpoints
- [`/docs/architecture/file-storage.md`](../../architecture/file-storage.md) — on-disk layout
- [QuyetDinh](./QuyetDinh.md), [HoSoSucKhoe](./HoSoSucKhoe.md) — possible owners
