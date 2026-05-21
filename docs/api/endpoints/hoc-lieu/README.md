# /api/hoc-lieu

Learning-materials repository. Folder/file tree across three independent drives (`deCuong`, `giaoTrinh`, `taiLieuThamKhao`). Metadata in Mongo (`HocLieu` model), binaries on disk under `backend/uploads/<drive>/`. Quotas: 3 GB per drive, 100 MB per file.

Roles: `admin`, `staff`, `viewer`, `teacher` (read + write for most). Per-unit scoping does NOT apply here — these are global resources, not student-bound.

**Source files**: [hocLieu.route.js](../../../../backend/src/routes/hocLieu.route.js), [hocLieu.controller.js](../../../../backend/src/controllers/hocLieu.controller.js), [hocLieu.service.js](../../../../backend/src/services/hocLieu.service.js), [HocLieu schema](../../../backend/schemas/HocLieu.md), [config/storage.js](../../../../backend/src/config/storage.js).

## Endpoints

### `GET /api/hoc-lieu`

List entries inside a drive/folder.

**Query**: `drive` (one of `deCuong` / `giaoTrinh` / `taiLieuThamKhao`; strongly recommended — no server-side validation enforces it; if omitted, returns items from **all drives**), `parent` (optional ObjectId; omit for drive root).

**Response**: `{ data: HocLieuItem[] }`. No `meta` — list is not paginated.

Each item: `{ _id, name, drive, type: 'file' | 'folder', parent, mimeType?, size?, storagePath?, uploadedBy?, createdAt, updatedAt }`.

### `GET /api/hoc-lieu/usage`

Per-drive bytes used + percentage of the 3 GB cap.

**Query**: `drive` (one of `deCuong` / `giaoTrinh` / `taiLieuThamKhao`; no server-side validation — if omitted, computes usage across all drives combined).

**Response**: `{ data: { used: <bytes>, limit: <bytes>, percentage: <0..100> } }`.

### `POST /api/hoc-lieu/folder`

Create a folder.

**Roles**: `admin`, `staff`, `teacher`.

**Body**: `{ name: string, drive: string, parent?: ObjectId | null }`. Validated by `createFolderSchema`.

**Response 201**: `{ data: HocLieuItem }`.

**Errors**: `400 VALIDATION_ERROR`, `404 PARENT_NOT_FOUND`, `409 DUPLICATE_NAME` (collision on `(drive, parent, name)`).

### `POST /api/hoc-lieu/upload`

Upload a file. `multipart/form-data`.

**Roles**: `admin`, `staff`, `teacher`.

**Fields**: `file` (required, ≤ 100 MB, must pass `ALLOWED_EXTENSIONS` + `ALLOWED_MIMETYPES`), `drive` (required), `parent` (optional).

**Response 201**: `{ data: HocLieuItem }`.

**Errors**: `400 INVALID_FILE_TYPE`, `404 PARENT_NOT_FOUND`, `409 DUPLICATE_NAME`, `413 DRIVE_FULL` (writing would exceed the 3 GB cap).

### `PATCH /api/hoc-lieu/:id/rename`

Rename a file or folder.

**Roles**: `admin`, `staff`, `teacher`.

**Body**: `{ name: string }`. Validated by `renameSchema`.

**Response**: `{ data: HocLieuItem }`.

**Errors**: `400`, `404 NOT_FOUND`, `409 DUPLICATE_NAME`.

> Folder rename only updates the folder's own `name` field. Descendant `storagePath` values use UUID names and are unaffected.

### `DELETE /api/hoc-lieu/:id`

Delete a file or recursively delete a folder.

**Roles**: `admin`, `staff`, `teacher`.

**Response**: `204 No Content`.

**Errors**: `404 NOT_FOUND`.

Cascade behavior: a folder delete removes all descendant Mongo rows AND their on-disk files. A file delete removes the Mongo row AND the on-disk file. No rollback if the disk delete fails after the Mongo delete.

### `GET /api/hoc-lieu/:id/download`

Stream a file to the client.

**Roles**: `admin`, `staff`, `viewer`, `teacher`.

**Response**: binary stream with `Content-Disposition: attachment; filename="<original-name>"` and the original `mimeType`. NOT a JSON envelope.

**Errors**: `404 NOT_FOUND` if the row is gone, the on-disk file is missing, OR the id refers to a folder (not a file).

## Cross-cutting notes

- **No per-unit scoping.** Learning materials are visible to every authenticated user with the right role; there is no `khoa`/`daiDoi` filter.
- **Unique-name-within-folder**: enforced by the Mongo `{ drive, parent, name }` unique index.
- **Quota math is application-level**: the upload route sums existing `size` values for the drive before writing the new file. There is no live FS check; if the host disk runs out, multer surfaces `ENOSPC` as a 500.
- **Allowed file types** are 30 extensions and 29 MIME types — Office docs, common images, audio/video, common archives. See [config/storage.js](../../../../backend/src/config/storage.js). Add new types there, not in the route.

## Related

- [HocLieu schema](../../../backend/schemas/HocLieu.md)
- [`docs/architecture/file-storage.md`](../../../architecture/file-storage.md) — on-disk layout and quotas
- [`docs/frontend/pages/quan-ly-giang-day/kho-hoc-lieu-so.md`](../../../frontend/pages/quan-ly-giang-day/kho-hoc-lieu-so.md) — the UI
