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
- **Attachments are 1-per-record.** Uploading a second scan to the same QĐ returns 409. Delete first.
- **`Loại` advanced filter** is **exact-match** against each student's *latest* decision (newest `ngayKiQD`). Students whose newest decision has a different `loaiQD` are excluded. The full decision set is loaded up-front so this latest-per-student join uses the actual newest, not a paginated subset. When the filter is set, the table iterates *decisions* (not the current paginated student page) and supplements off-page students via `getMany` — so every match shows up regardless of which student page the user is on. Clearing the filter drops the supplement set and the table falls back to the standard server-paginated student list.
