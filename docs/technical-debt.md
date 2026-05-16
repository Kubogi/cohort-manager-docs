# Technical Debt

Known limitations identified during the documentation review process. Each item is a gap between what the code intends and what it actually enforces. Ordered by impact.

---

## TD-001 — User `status` field is half-implemented

**File**: `backend/src/validators/user.validator.js`, `backend/src/services/user.service.js`

The `User` model has a `status` field (`'active'` | `'disabled'`) and `authGuard` correctly blocks login for `status === 'disabled'` users. However, `updateUserSchema` accepts only `role`, `password`, `allowedUnits`, and `teacherScope` — no `status` field. `updateUser()` in the service also has no `status` parameter.

**Impact**: There is currently no API path to disable a user. The disable check in `authGuard` is dead code.

**Fix**: Add `status: Joi.string().valid('active', 'disabled')` to `updateUserSchema` and propagate it through `updateUser()`.

---

## TD-002 — `sinhVien.service.js` temp file not cleaned up on exception

**File**: `backend/src/services/sinhVien.service.js`

The student import service calls `fs.unlink(filePath)` at the end of the function body without a `try/finally` wrapper. If an exception is thrown during processing (e.g., `DaiDoi.create()` or `SinhVien.insertMany()` fails), the temp file in `backend/uploads/.tmp/` is orphaned until the cleanup cron runs.

**Impact**: Failed imports silently leave temp files on disk. With large Excel files this can accumulate significant disk usage.

**Fix**: Wrap the import logic in `try/finally` and move the `fs.unlink` call into the `finally` block, matching the pattern used by `canBoQuanLy.controller.js` and `diem.controller.js`.

---

## TD-003 — Deleting a health record orphans its attachment

**File**: `backend/src/services/hoSoSucKhoe.service.js`

`hoSoSucKhoe.service.remove()` calls `findOneAndDelete()` on the health record but never calls the attachment service to delete the associated `Attachment` row or its on-disk file. The `HoSoSucKhoe` schema has an `attachment` field referencing an `Attachment` document.

**Impact**: Every deleted health record leaves an orphaned `Attachment` DB row and a file on disk under `backend/uploads/ho-so-suc-khoe/`. These accumulate silently.

**Fix**: In `hoSoSucKhoe.service.remove()`, after `findOneAndDelete`, check if the deleted document had an `attachment` field and call the attachment removal logic (delete DB row + unlink file) for it.

---

## TD-004 — QĐ processing → student status sync is frontend-only

**File**: `backend/src/services/quyetDinh.service.js`

The intended invariant is: processing a `QuyetDinh` (setting `trangThai: 'Đã xử lý'`) should update the linked student's `trangThai`; un-processing should revert it to `'Đang học'`. The frontend implements this as two sequential PATCH calls. The backend `quyetDinh.service.update()` function only writes the QĐ document — it never touches `SinhVien`.

**Impact**: Any direct API caller (scripts, Postman, other services) that PATCHes a QĐ's `trangThai` will change the decision record without updating the student, silently breaking the invariant. No validation or error is raised.

**Fix** (options):
- Move the sync logic into `quyetDinh.service.update()` as an atomic side effect.
- Or add a dedicated `PATCH /api/quyet-dinh/:id/process` endpoint that handles both writes in a single transaction.
