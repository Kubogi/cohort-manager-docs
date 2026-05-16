# Learning Materials

**Triggered from**: [Kho học liệu số](../frontend/pages/quan-ly-giang-day/kho-hoc-lieu-so.md).

**Touches**: `/api/hoc-lieu/*` (folder + file + usage + download endpoints), `HocLieu` model, `backend/uploads/<drive>/` on disk.

**Who can do this**: `admin`, `staff`, `teacher` for writes; `admin`/`staff`/`viewer`/`teacher` for reads.

## Goal

Browse, upload, organize, and download teaching materials across three independent drives (`giaoAn`, `giaoTrinh`, `taiLieuThamKhao`), each with a 3 GB quota.

## Happy path — upload a file

1. User opens **Kho học liệu số**, picks the drive (Giáo án / Giáo trình / Tài liệu tham khảo).
2. Navigates to the destination folder via the breadcrumb.
3. Clicks **Tải lên**, selects a file from the OS picker.
4. Frontend sends `POST /api/hoc-lieu/upload` as `multipart/form-data` with `file`, `drive`, `parent` (the current folder id or null for root).
5. Backend:
   - Multer enforces `≤ 100 MB` and MIME type (via `ALLOWED_MIMETYPES` in the `fileFilter`). Extension validation using `ALLOWED_EXTENSIONS` is a **separate post-multer guard in `hocLieu.service.js`** — multer does not check extensions.
   - Service confirms `parent` exists, computes drive usage, rejects if quota would be exceeded (`413 DRIVE_FULL`).
   - Writes the file under `uploads/<drive>/<path>/<sanitized-name>`.
   - Inserts a `HocLieu` row with `type: 'file'`, `storagePath`, `mimeType`, `size`, `uploadedBy: req.user.id`.
6. Frontend appends the new row to the listing.

## Happy path — create folder tree

1. **Tạo thư mục** at the current location. Enter the name, save → `POST /api/hoc-lieu/folder`.
2. Open the new folder. **Tạo thư mục** again for sub-folders.
3. Folder rows are pure metadata — no on-disk directory is created until the first file lands inside.

## Happy path — download

1. Click a file row's download action → `GET /api/hoc-lieu/:id/download`.
2. Backend streams the bytes with `Content-Disposition: attachment`.

## Happy path — rename

`PATCH /api/hoc-lieu/:id/rename` with `{ name }`. The `(drive, parent, name)` unique index enforces no collisions; 409 on conflict.

## Happy path — delete

1. Click **Xoá**. Confirmation modal: if it's a folder, lists the descendant count.
2. `DELETE /api/hoc-lieu/:id`.
3. Service walks `{ parent: id, drive }` recursively, deletes each child row and its on-disk path, then the root.
4. Browser refresh: row gone.

## Side-effects

- **Drive usage updates implicitly.** The `usage` endpoint computes `sum(size)` per drive on demand; no cached counter.
- **Orphan files possible** if the FS write fails after the Mongo write (or vice versa). No rollback. Periodic reconciliation (script that diffs Mongo entries against disk paths) is on the roadmap but not implemented.

## Failure modes

| Scenario | Result |
|---|---|
| File extension / MIME not in allow-list | `400 INVALID_FILE_TYPE` |
| File > 100 MB | Multer surfaces as a 5xx via the global handler; UX hint: split the archive. |
| Folder already contains a file/folder with this name | `409 DUPLICATE_NAME` |
| Drive at quota | `413 DRIVE_FULL` |
| Parent folder ID doesn't exist (UI stale) | `404 PARENT_NOT_FOUND` |
| Disk full at OS level | `500 INTERNAL_ERROR` (`ENOSPC`) |

## Manual test recipe

- [ ] Upload a 50 MB PDF into Giáo án root. Confirm the row appears.
- [ ] Create a sub-folder, move the upload (via a follow-up upload or future move endpoint).
- [ ] Try uploading a `.exe` — expect `400 INVALID_FILE_TYPE`.
- [ ] Check `GET /api/hoc-lieu/usage?drive=giaoAn` matches what `du -sh backend/uploads/giaoAn` reports (close enough; HocLieu rows track logical bytes, FS tracks block-aligned bytes).
- [ ] Delete the sub-folder; verify the on-disk path is gone too.
