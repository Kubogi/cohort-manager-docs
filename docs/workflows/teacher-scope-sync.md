# Teacher-scope auto-sync from CanBoQuanLy.phanCong[]

**Service**: [`backend/src/services/teacherScope.service.js`](../../backend/src/services/teacherScope.service.js) — `syncTeacherScope(userId)`
**Last verified**: 2026-05-19

---

## Why

A `User` with `role: 'teacher'` is typically linked to a `CanBoQuanLy` (via `CanBoQuanLy.taiKhoan`). The CBQL already tracks which khoa + daiDoi pairs that person manages, via `phanCong[]`. Keeping the User's `teacherScope` in sync with the CBQL's `phanCong` means admins don't have to maintain the same data in two places — and a teacher's visible students automatically follow their assignment changes.

The User's `teacherScope` is split into two arrays:
- `teacherScope[]` — **admin-managed**. Admin sets these in the user-management UI ("Cấu hình phạm vi truy cập" popup). Survives sync untouched.
- `teacherScopeSynced[]` — **system-managed**. Rewritten by the sync engine.

[`applyTeacherScope`](../../backend/src/utils/permissions.js) reads the union; permission semantics are unchanged.

---

## What gets synced

For the User linked to a CBQL, the sync engine walks `cbql.phanCong[]`:

| phanCong entry | Synced as |
|---|---|
| `{khoa: X, daiDoi: Y}` | `{khoa: X, daiDoi: Y}` |
| `{khoa: X, daiDoi: null}` (orphan / khoa-only) | **skipped** — no scope entry produced |

The orphan-skip rule matches the user's design intent ("khoa+daiDoi pairs"). A CBQL whose only assignment to a khoa is orphan grants the linked teacher no scope in that khoa.

If the linked User is **not** `role: 'teacher'` (admin, staff, viewer), sync clears `teacherScopeSynced` to `[]` — non-teachers never receive synced scope.

If a `User` has **no** linked CBQL (no CBQL.taiKhoan points at them), `teacherScopeSynced` is cleared to `[]`.

---

## Triggers

The sync runs automatically whenever the underlying data could shift:

| Trigger | File | What it does |
|---|---|---|
| `POST /api/can-bo-quan-ly` (create) | [`canBoQuanLy.service.js`](../../backend/src/services/canBoQuanLy.service.js) `create` | If `taiKhoan` is set on the new CBQL, `syncTeacherScope(taiKhoan)`. |
| `PATCH /api/can-bo-quan-ly/:id` (update) | same file `update` | Snapshots the old `taiKhoan`. If `taiKhoan` or `phanCong` is in the payload: re-sync the new `taiKhoan` (if set), and the old `taiKhoan` (if different — clears its now-stale synced entries). |
| `DELETE /api/can-bo-quan-ly/:id` | same file `remove` | If the deleted CBQL had a `taiKhoan`, re-sync that user (synced becomes `[]`). |
| `POST /api/can-bo-quan-ly/import` (Excel bulk import) | same file `importFromExcel` | Collects affected userIds across the loop, runs `Promise.all` of `syncTeacherScope` after all writes settle. |
| `DELETE /api/khoa-hoc/:id` | [`khoa.service.js`](../../backend/src/services/khoa.service.js) `remove` | Before pulling matching phanCong entries, snapshots every CBQL with a linked `taiKhoan` and a phanCong entry for this khoa. After the `$pull`, re-syncs each user. |
| `DELETE /api/don-vi/dai-doi/:id` | [`donVi.service.js`](../../backend/src/services/donVi.service.js) `removeDaiDoi` | Before demoting phanCong entries (`$set phanCong.$[el].daiDoi: null`), snapshots affected linked users; re-syncs them after — the demoted (now orphan) entries are skipped during sync, so the user loses scope for that daiDoi. |

---

## How admin manual additions survive

The split-storage design means manual entries are physically separate from synced entries. The admin PATCH endpoint only accepts `teacherScope` in the payload (Joi `stripUnknown` drops `teacherScopeSynced` silently). The sync engine only writes to `teacherScopeSynced`. There is no overlap path.

This means:
- Admin can grant a teacher extra scope (e.g. `{khoa: K5, allDaiDoi: true}`) — it lives in `teacherScope[]` and is never touched by sync.
- Removing a CBQL.phanCong entry only removes the corresponding synced entry, never a manual one.
- Re-syncing is safe: synced entries are replaced wholesale, manual entries stay put.

---

## UI surface

The "Cấu hình phạm vi truy cập" popup (Quản lý tài khoản → Sửa user → button) renders both arrays in a single table:
- Synced rows appear **first**, greyed out (`opacity: 0.55` + muted background) with a small "Đồng bộ" badge. Their delete button is disabled with a tooltip pointing to the khoaHoc page.
- Manual rows render normally below — editable and deletable.
- The hint *"Các dòng được tô mờ tự động đồng bộ từ phân công CBQL — chỉnh sửa qua trang Khóa học."* appears above the table when any synced rows exist.

Save sends only the manual entries; synced rows are display-only.
