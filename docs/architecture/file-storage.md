# File Storage

User-uploaded files (attachments to records, learning materials, Excel imports) live on the API host's local filesystem under `backend/uploads/`. Read-only Excel templates committed to git live under `backend/forms/`. There is no object storage, no CDN, no shared volume — every file is written to and read from the same disk that runs the Express process.

This document covers the on-disk layout, the per-route size/type limits, the multer wiring, and the operational follow-ups (backups, quota monitoring) that the in-code limits don't handle.

## 1. On-disk layout

```
backend/
├─ forms/                         ── Excel templates (read-only, committed to git)
│  ├─ Hệ ĐH/                      ── Đại học (university) templates
│  │  ├─ DS điểm danh.xlsx
│  │  ├─ DS điểm Thường xuyên.xlsx
│  │  ├─ DS điểm giữa kỳ.xlsx
│  │  ├─ DS điểm hết HP.xlsx
│  │  ├─ Sổ điểm từng môn.xlsx
│  │  └─ Sổ tổng.xlsx              ── Bound summary template (used by /api/diem/export-so-tong; not in TEMPLATE_MAP)
│  ├─ Hệ CĐ/                      ── Cao đẳng (college) templates
│  │  ├─ DS điểm danh.xlsx
│  │  ├─ DS điểm miệng.xlsx
│  │  ├─ DS điểm thường xuyên.xlsx
│  │  ├─ DS điểm giữa HP.xlsx
│  │  ├─ DS điểm hết HP.xlsx
│  │  ├─ Sổ điểm các môn.xlsx
│  │  └─ Sổ tổng.xlsx              ── Bound summary template (used by /api/diem/export-so-tong; not in TEMPLATE_MAP)
│  ├─ khao-sat-chat-luong/        ── Survey templates (not currently wired to an endpoint)
│  │  ├─ phieu-gv-tu-danh-gia-2024-2025.xlsx
│  │  ├─ phieu-phan-hoi-nguoi-hoc-ve-gv-2024-2025.xlsx
│  │  └─ phieu-phan-hoi-nguoi-hoc-ve-trung-tam-2024-2025.xlsx
│  └─ Danh sách GV.xlsx           ── Instructor list (used as import source)
│
└─ uploads/                       ── User-generated files (NOT in git; back up separately)
   ├─ .tmp/                       ── Multer staging for Excel imports — deleted after parse
   ├─ attachments/                ── Per-record persistent attachments
   │  ├─ QuyetDinh/<uuid>.<ext>   ── one attachment per QuyetDinh
   │  └─ HoSoSucKhoe/<uuid>.<ext> ── one attachment per HoSoSucKhoe
   ├─ deCuong/                    ── Học liệu drive 1 (syllabi / course outlines)
   ├─ giaoTrinh/                  ── Học liệu drive 2 (coursebooks)
   └─ taiLieuThamKhao/            ── Học liệu drive 3 (reference materials)
```

The three `học liệu` drive folders contain a tree that mirrors the `HocLieu` collection's folder hierarchy. Each `HocLieu` document's `storagePath` field is the on-disk path relative to `uploads/`.

`UPLOADS_ROOT` is resolved by `backend/src/config/storage.js`:

```js
export const UPLOADS_ROOT = path.resolve(__dirname, '../../uploads');
```

So the uploads tree always lives next to the `backend/src/` source tree. In production, **`backend/uploads/` is your stateful data** alongside MongoDB. If you redeploy by `git clone`-ing into a new directory, you must move or symlink this folder, or the existing uploads become inaccessible.

## 2. Configured limits

From `backend/src/config/storage.js`:

| Limit | Value | Enforced where |
|---|---|---|
| `MAX_DRIVE_SIZE` | 3 GB per drive (`deCuong`, `giaoTrinh`, `taiLieuThamKhao`) | `hocLieu.service.upload` before write |
| `MAX_FILE_SIZE` | 100 MB per uploaded file | multer config in `hoc-lieu` route |
| Excel import file size | 20 MB | multer config in `sinh-vien` + `diem` routes |
| `ALLOWED_EXTENSIONS` (30 entries) | `.pdf .doc .docx .xls .xlsx .ppt .pptx .txt .csv .rtf .odt .ods .odp .epub .jpg .jpeg .png .gif .bmp .svg .webp .mp3 .mp4 .wav .avi .mkv .webm .zip .rar .7z` | Checked in `hocLieu.service.js` as a post-multer extension guard — actively used, not dead code. Multer's fileFilter checks MIME type at the route level; the service separately checks the extension against this list. |
| `ALLOWED_MIMETYPES` (29 entries) | matching Office / image / audio / video / archive MIME types | multer file filter for hoc-lieu and attachments |

The 3 GB drive quota is **soft** — `hocLieu.service.upload` checks the existing on-disk usage before writing the new file. There is no live disk-space check; if the filesystem fills before Mongo records reach 3 GB, writes fail at the OS level (multer surfaces `ENOSPC`).

## 3. Multer wiring (per route)

| Route | Storage | Destination | Filename | Size limit | File filter |
|---|---|---|---|---|---|
| `POST /api/sinh-vien/import` | `diskStorage` | `uploads/.tmp/` | `<uuid>.<ext>` | 20 MB | extension must be `.xls`/`.xlsx` |
| `POST /api/diem/import` | `diskStorage` | `uploads/.tmp/` | `<uuid>.<ext>` | 20 MB | extension must be `.xls`/`.xlsx` |
| `POST /api/khao-sat-chat-luong/process` | `diskStorage` | `uploads/.tmp/` | `<uuid>.<ext>` | 20 MB | extension must be `.xls`/`.xlsx` |
| `POST /api/hoc-lieu/upload` | `diskStorage` | `uploads/.tmp/` then moved by service | sanitized original | 100 MB (`MAX_FILE_SIZE`) | `ALLOWED_MIMETYPES` at route — no extension check at filter level |
| `POST /api/attachments` | `diskStorage` | `uploads/.tmp/` then moved by service | sanitized original | 100 MB (`MAX_FILE_SIZE`) | `ALLOWED_MIMETYPES` at route |
| `POST /api/can-bo-quan-ly/import` | `diskStorage` | `uploads/.tmp/` | `<uuid>.<ext>` | 20 MB | extension must be `.xls`/`.xlsx` |

After multer writes the file to disk, the controller takes over:

- **`sinh-vien` Excel import** parses the file with `xlsx` (SheetJS) and calls `fs.unlink(filePath)` at the end of the function body **without a try/finally wrapper**. If an exception is thrown before that line (e.g., during `DaiDoi.create` or `SinhVien.insertMany`), the temp file is orphaned until the cleanup cron runs. `.tmp/` should normally be empty.
- **`diem` Excel import** parses with `exceljs`, validates row-by-row, and saves in parallel chunks of 50. Per-row errors are returned in the response envelope. Temp file deleted in `finally`.
- **`khao-sat-chat-luong/process`** parses with `xlsx` (SheetJS) — see [`khaoSatChatLuongProcess.service.js`](../../backend/src/services/khaoSatChatLuongProcess.service.js). Temp file deleted in `finally`.
- **Học liệu uploads** create or update a `HocLieu` Mongo document with `storagePath`, `size`, `mimeType`, and `uploadedBy`, then leave the file in its final location under `uploads/<drive>/<folder-path>/`.
- **Attachments** create an `Attachment` document with `ownerType`/`ownerId`/`storagePath`. The unique index on `{ ownerType, ownerId }` means **one attachment per record** — uploading a second to the same QuyetDinh overwrites (or, depending on the controller, 409s).

## 4. Reading files back

| Route | Behavior |
|---|---|
| `GET /api/hoc-lieu/:id/download` | Streams the file from `UPLOADS_ROOT + storagePath`. Sets `Content-Disposition: attachment` with the original `name`. Auth required. |
| `GET /api/attachments/:id/download` | Same pattern for attachments. |
| Static serving | **None.** Nginx is not configured to serve from `backend/uploads/`; all reads go through Express so the `authGuard` runs. Don't add `app.use('/uploads', express.static(...))` — it bypasses auth. |

## 5. Failure modes you'll actually see

| Status | Code | Where it comes from |
|---|---|---|
| 400 | `INVALID_FILE_TYPE` | hoc-lieu upload, extension or MIME mismatch |
| 400 | `INVALID_FILE` | sinh-vien import, no file or not `.xls/.xlsx` |
| 404 | `PARENT_NOT_FOUND` | hoc-lieu upload to a non-existent parent folder |
| 409 | `DUPLICATE_NAME` | hoc-lieu upload colliding with `{ drive, parent, name }` unique index |
| 413 | `DRIVE_FULL` | hoc-lieu upload that would push the drive past 3 GB |
| 413 (multer) | — | File exceeds `limits.fileSize` (handled by `errorHandler` as 500 unless intercepted) |
| 500 | `INTERNAL_ERROR` | Disk full (`ENOSPC`), permission denied (`EACCES`), or the file was deleted between Mongo write and serve |

If multer rejects a file at the boundary, Express may emit `MulterError`. The current error envelope does not specifically translate these — they pass through `errorHandler` as 500. Consider adding a dedicated branch if multer errors become user-visible.

## 6. Operational concerns

### Backups

`backend/uploads/` is **not** in `mongodump`. To back up the system end-to-end you need both:

```bash
mongodump --uri="$MONGO_URI"     --out=/backups/mongo-primary/$(date +%F)
mongodump --uri="$MONGO_URI2"    --out=/backups/mongo-secondary/$(date +%F)
rsync -a /var/www/backend/uploads/  /backups/uploads/$(date +%F)/
```

Restore order: Mongo first, then sync the files. If the Mongo document references a file that's no longer on disk, downloads 404 silently — there's no consistency check.

### Disk monitoring

The 3 GB per-drive cap is enforced in code, but the host filesystem has no enforced limit. Recommended:

- Allocate a separate partition or volume for `backend/uploads/` (or symlink it to a managed volume).
- Set a host-level monitor (`df -h /var/www/backend/uploads`) — alert at 80 % full.
- The three drives' caps total 9 GB; allocate at least 15 GB to absorb attachments and the `.tmp/` staging area.

### Cleanup of `.tmp/`

Most Excel-import controllers delete their staging files on success and on failure via `finally` blocks (`canBoQuanLy.controller.js`, `diem.controller.js`, and `khaoSatChatLuongProcess.service.js`). `sinhVien.service.js` does not use a `finally` block — its `fs.unlink` call is at the end of the function body, so a mid-import exception will orphan the file. If the process is killed mid-import (PM2 reload, server crash), orphan files remain in any case. Run a periodic cleanup:

```bash
find backend/uploads/.tmp -type f -mmin +60 -delete
```

This is safe — `.tmp/` is only ever used during a single request.

### File-system permissions

The Node process needs read+write under `backend/uploads/`. In production with PM2 running as a non-root user:

```bash
chown -R deploy:deploy backend/uploads
chmod -R u+rwX,g+rX,o-rwx backend/uploads
```

`backend/forms/` is read-only — keep it read-only on disk too (`chmod -R a-w backend/forms`) so the process can't accidentally corrupt templates.

### Migrating to object storage (future)

Today every upload is filesystem-local, which couples the API server to its disk. If you ever scale to multiple Node instances, you'll need to swap the storage backend. The path forward:

1. Replace `multer.diskStorage` with `multer-s3` (or equivalent) in each upload route.
2. Change `HocLieu.storagePath` semantics from a filesystem path to an object key.
3. Replace the `fs.createReadStream` in download handlers with a presigned URL redirect (or proxied stream from S3 with auth still in front).

The current schema is already mostly opaque about the storage scheme — `storagePath` is just a string. The blockers are: the disk-based size-checking in `hocLieu.service.upload`, and the `multer.diskStorage` calls in each route.

## 7. See also

- [`backend.md`](backend.md) — multer's place in the request lifecycle
- [`excel-pipeline.md`](excel-pipeline.md) — how `backend/forms/` feeds `/api/bieu-mau/export`
- [`database.md`](database.md) — `HocLieu` and `Attachment` schema details
- `backend/src/config/storage.js` — the canonical list of limits and allowed types
- [`docs/guides/backups.md`](../guides/backups.md) — backup/restore runbook
