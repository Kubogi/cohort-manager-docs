# Quản lý các QĐ/CV — Decisions & memoranda

**Menu path**: Quản lý sinh viên › Quản lý các QĐ/CV
**Roles**: admin · staff · viewer · teacher *(teachers see only decisions for students within their `teacherScope`; write actions remain admin-only — teachers cannot Thêm QĐ or import)*
**Source files**: [`frontend/src/resources/quanLySinhVien/quanLyCacQuyetDinh.tsx`](../../../../frontend/src/resources/quanLySinhVien/quanLyCacQuyetDinh.tsx) + companion folder `quyetDinh/`
**Related API endpoints**: [`/api/quyet-dinh/*`](../../../api/README.md#decisions-apiquyet-dinh), [`/api/sinh-vien/:id`](../../../api/endpoints/sinh-vien/patch.md), [`/api/attachments/*`](../../../api/README.md#attachments-apiattachments)
**Related workflows**: **[`decision-processing.md`](../../../workflows/decision-processing.md)** — required reading

## When to use

Manage administrative decisions (Hoãn học, Thôi học, Đình chỉ, Miễn học) and the status transitions they cause on the linked student. Every change to a student's `trangThai` away from "Đang học" goes through a quyết định on this page.

## Layout

- Filter bar: `khoa`, `đại đội`, `loại QĐ`, `trạng thái`.
- Advanced search: `maSV`, `hoTen`, `ngaySinh`, `lop`, `soQD` (substring, case-insensitive), `ngayKiQD` (exact day, dd/mm/yyyy), `loaiQD`, `trangThai`. The three decision-side filters (`loaiQD`, `soQD`, `ngayKiQD`) join on each student's **latest** decision via `latestDecisionByStudentId`; setting ANY of them activates the cross-page supplement path, so a matching student appears regardless of which server-paginated student page they live on.
- Action buttons: **Thêm QĐ** | **Xuất Excel**.
- Table: paginated list of `QuyetDinh` rows joined with the student.

## Common tasks

### Create an unprocessed QĐ

Click **Thêm QĐ**, pick the student, fill `soQD`, `ngayKiQD`, `loaiQD`, `lyDo`, optionally attach a scan. Save. The QĐ lands with `trangThai: "Chưa xử lý"`; no student-status change.

### Apply a QĐ to a student

Open the row, edit, flip `trangThai` to "Đã xử lý". A confirmation modal appears warning that the student's status will change. Confirm → student's `trangThai` flips to match `loaiQD`. See [`decision-processing.md`](../../../workflows/decision-processing.md).

### Revert a processed QĐ

Same path, flip `trangThai` back to "Chưa xử lý". Confirmation modal warns the student will revert to "Đang học".

### Delete a processed QĐ

Click **Xoá**. Warning modal: "This will revert the student to Đang học." Confirm → frontend first reverts the student status, then deletes the QĐ.

## Edge cases / gotchas

- **Two-step save for status changes.** The frontend never bundles `trangThai` change + side-effect into one PATCH; it always splits across separate requests with a confirmation modal in between. This is non-obvious but intentional — see the workflow doc.
- **Delete of an unprocessed QĐ is a simple confirmation.** No student-status modal because there's no side-effect.
- **Cross-referenced student must exist.** If the linked `sinhVien` ObjectId is missing (deleted in error), processing the QĐ silently no-ops on the student but still flips the QĐ status. Inconsistent state requires manual fix.
- **Edit modal locks `maSV`.** When editing an existing QĐ, the `Mã SV` field is prefilled from the joined student record and rendered disabled — you cannot move a QĐ to a different student by retyping the mã. (Create-for-student also locks it; only the free-form Thêm QĐ flow allows typing.) The lock + prefill flows through `studentById`, which merges the off-page `loaiQdSupplements` so the field still populates correctly when the edited student lives off the current paginated student page (e.g. opened from a `loaiQD`-filtered view).
- **Attachments are 1-per-record.** Uploading a second scan to the same QĐ returns 409. Delete first.
- **`Loại` advanced filter** is **exact-match** against each student's *latest* decision (newest `ngayKiQD`). Students whose newest decision has a different `loaiQD` are excluded. The full decision set is loaded up-front so this latest-per-student join uses the actual newest, not a paginated subset. When the filter is set, the table iterates *decisions* (not the current paginated student page) and supplements off-page students via `getMany` — so every match shows up regardless of which student page the user is on. The supplement is also folded into `studentById`, so write paths triggered from the filtered view (edit/save, process, unprocess, delete-with-revert) resolve the off-page student correctly instead of silently no-op'ing on the student-side update. Clearing the filter drops the supplement set and the table falls back to the standard server-paginated student list.
- **Pagination counter is filter-aware.** When any decision-side filter (`loaiQD` / `soQD` / `ngayKiQD`) is active the `Tổng` header reads the number of matched candidates rather than the server-side student total, and the rendered rows are sliced by the current `page` × `pageSize`. Without this, the header reported the unfiltered khóa headcount (e.g. "Tổng: 312") while the body showed 3 rows — or zero rows when no decisions matched — making it look like the filter had silently broken. Clearing the filter reverts the counter to the server-side total.
- **`Khoa` / `Đại đội` filter requires resolution through `SinhVien`.** The backend `GET /api/quyet-dinh?khoa=<id>` resolves the khoa through the student collection (`SinhVien.khoa` → `QuyetDinh.sinhVien $in`) rather than filtering `QuyetDinh.khoa` directly. The `QuyetDinh` schema has `khoa` + `daiDoi` fields but the create/update payload never populates them, so a direct field filter returned zero rows — visible to the user as an empty table the moment a `Khoa` was picked at the top of the page. Staff/teacher scope is intersected with the resolved khoa student set so the gating still applies.
