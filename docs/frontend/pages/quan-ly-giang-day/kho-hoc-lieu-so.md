# Kho học liệu số — Digital learning materials repository

**Menu path**: Quản lý giảng dạy › Kho học liệu số

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/quanLyGiangDay/khoHocLieuSo/khoHocLieuSo.tsx`](../../../../frontend/src/resources/quanLyGiangDay/khoHocLieuSo/khoHocLieuSo.tsx) + companion folder

**Related API endpoints**: [`/api/hoc-lieu/*`](../../../api/README.md#learning-materials-apihoc-lieu)

**Related workflows**: [`learning-materials.md`](../../../workflows/learning-materials.md)

## When to use

Upload, organize, and download teaching materials. The repository is organized into **three drives** (`giáo án`, `giáo trình`, `tài liệu tham khảo`); each is a self-contained folder tree with a 3 GB quota.

## Layout

- Drive switcher at the top: **Giáo án** | **Giáo trình** | **Tài liệu tham khảo**.
- Breadcrumb showing the current path within the active drive.
- Toolbar: **Tạo thư mục** | **Tải lên** | drive-usage indicator (e.g. `1.2 GB / 3 GB`).
- Two-pane explorer-style view: folder tree on the left, file/folder list on the right.
- Right-click (or row hover) menu per item: **Đổi tên** | **Tải về** (files only) | **Xoá**.

## Common tasks

### Upload a file into a specific folder

1. Switch to the right drive.
2. Navigate to the destination folder (or stay at root).
3. **Tải lên**, pick a file. Multer accepts up to 100 MB per file and validates against `ALLOWED_EXTENSIONS` + `ALLOWED_MIMETYPES`.
4. The new file appears in the list with name, size, mimeType.

### Create a folder hierarchy

1. **Tạo thư mục** at the current location. Type name, save.
2. Open the new folder, **Tạo thư mục** again for sub-folders.

### Rename a folder

1. Hover a folder row, **Đổi tên**. Type new name. Backend enforces uniqueness within `(drive, parent)`.

### Delete a folder with contents

Confirmation modal lists the children that will also be deleted. On confirm, the service recursively removes child rows in Mongo and then the on-disk files.

## Edge cases / gotchas

- **3 GB per drive is enforced at upload time.** Trying to upload a file that would push the drive past 3 GB returns 413 `DRIVE_FULL`. Free up space first.
- **Unique-name-per-folder.** Two files with the same name in the same `(drive, parent)` aren't allowed. The error is 409 `DUPLICATE_NAME`.
- **Allowed extensions are fixed.** See [`storage.js`](../../../../backend/src/config/storage.js) for the list (PDF, Office, images, audio/video, archives — no `.exe`, `.app`, etc.).
- **Subtree rename behavior is implementation-dependent.** Confirm that renaming a folder also updates the `storagePath` of all descendants before doing it on a large subtree. If unsure, test on a copy.
- **Deletion cascades on disk.** A successful delete removes the on-disk file in the same operation. If the FS write fails after the Mongo delete, the orphan file is lost — there's no rollback.
- **Teacher can access this page** even though "Quản lý sinh viên" is hidden from them. Teaching materials are part of the teacher's daily work.
