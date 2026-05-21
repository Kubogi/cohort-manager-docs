# HocLieu Schema

**Source**: [backend/src/models/HocLieu.js](../../../backend/src/models/HocLieu.js)

**Collection**: `hoc_lieu` (primary cluster)

**Last verified**: 2026-05-20

A node in the learning-materials repository. Each row is either a **file** or a **folder**. Folders form a tree via the self-referential `parent` field. Three independent trees coexist, one per **drive** (`deCuong`, `giaoTrinh`, `taiLieuThamKhao`); a node's `(drive, parent)` pair uniquely scopes it.

> Drive `deCuong` was renamed from `giaoAn` on 2026-05-20 (label: "Giáo án" → "Đề cương"). Existing documents were migrated via [`backend/src/scripts/rename-giaoAn-to-deCuong.js`](../../../backend/src/scripts/rename-giaoAn-to-deCuong.js). The `giaoAn` value is no longer accepted by the enum.

The actual file contents are stored on disk under `backend/uploads/<drive>/...`; this collection holds only the metadata. See [`/docs/architecture/file-storage.md`](../../architecture/file-storage.md).

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | String | Yes | — | Display name. Unique within `(drive, parent)`. |
| `drive` | String enum | Yes | — | One of `'deCuong'`, `'giaoTrinh'`, `'taiLieuThamKhao'` (sourced from `DRIVE_NAMES` in `config/storage.js`). Top-level partition. |
| `type` | String enum | Yes | — | `'file'` or `'folder'`. |
| `parent` | ObjectId → HocLieu | No | `null` | Parent folder (self-reference). `null` means root of the drive. |
| `mimeType` | String | No | — | Set for files; copied from multer at upload time. |
| `size` | Number (bytes) | No | — | Set for files. Folders are 0 / unset. |
| `storagePath` | String | No | — | Disk path relative to `UPLOADS_ROOT`. Set for files only. |
| `uploadedBy` | ObjectId → User | No | — | User who uploaded the file (null for folders). |
| `createdAt`, `updatedAt` | Date | Auto | — | Mongoose timestamps. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ drive: 1, parent: 1, name: 1 }` | unique | One node per name within a folder + drive |
| `{ drive: 1, parent: 1 }` | non-unique | Fast listing of a folder's children |

## Behavior notes

- **Deleting a folder** is a recursive operation: the service walks `{ parent: <folderId>, drive: <drive> }` and removes children before the folder itself, then deletes the on-disk paths in the same order. There's no foreign-key cascade — the service does it manually.
- **Renaming** updates `name` and recomputes `storagePath` for the renamed node and (in some implementations) its descendants. Check `services/hocLieu.service.js` for the current behavior before relying on subtree-rename semantics.
- **Per-drive quota** (3 GB, from [`storage.js`](../../../backend/src/config/storage.js)) is checked at upload time only — the service sums the existing `size` values for the drive and rejects if the new file would exceed the cap. There is no live disk-space check.
- **Allowed file types** are enforced by multer's `fileFilter` using `ALLOWED_EXTENSIONS` and `ALLOWED_MIMETYPES` from `storage.js`.

## Related

- [`/api/hoc-lieu/*`](../../api/README.md#learning-materials-apihoc-lieu) — endpoints
- [`/docs/architecture/file-storage.md`](../../architecture/file-storage.md) — on-disk layout + quotas
- [`/docs/workflows/learning-materials.md`](../../workflows/learning-materials.md) — upload + browse flow
