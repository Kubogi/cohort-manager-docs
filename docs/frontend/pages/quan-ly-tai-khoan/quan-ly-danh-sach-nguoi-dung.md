# Quản lý danh sách người dùng — User management

**Menu path**: Quản lý tài khoản › Quản lý danh sách người dùng
**Roles**: admin · staff · viewer *(not visible to teacher)* — most write actions are **admin-only** at the route level
**Source files**: [`frontend/src/resources/quanLyTaiKhoan/quanLyDanhSachNguoiDung/quanLyDanhSachNguoiDung.tsx`](../../../../frontend/src/resources/quanLyTaiKhoan/quanLyDanhSachNguoiDung/quanLyDanhSachNguoiDung.tsx) + companion folder
**Related API endpoints**: [`/api/users/*`](../../../api/README.md#users-apiusers), [`POST /api/auth/register`](../../../api/endpoints/auth/post-register.md)
**Related workflows**: [`user-management.md`](../../../workflows/user-management.md)
**Related manual playbook**: [`/planning/TESTING_GUIDE_USER_MANAGEMENT.md`](../../../../planning/TESTING_GUIDE_USER_MANAGEMENT.md)

## When to use

Admin creates, edits, disables, or deletes user accounts. Configures each user's `role`, `allowedUnits` (staff/viewer scope), and `teacherScope` (teacher scope).

## Layout

- Filter bar: search by `username` only — there is no role filter dropdown.
- Action buttons: **Thêm người dùng** only — there is no "Xuất Excel" button.
- Table columns: **username**, **role**, **actions** (edit/delete buttons). The columns `allowedUnits`, `status`, and `lastLogin` do not exist in the table.

## Common tasks

### Create a teacher with a specific scope

1. **Thêm người dùng** opens a modal.
2. Fill `username`, `password`, set `role = teacher`.
3. The modal expands to show the **"Cấu hình phạm vi truy cập"** button. Clicking it opens the TeacherScopeEditor modal (820px). The editor renders two kinds of rows:
   - **Synced rows** (top, greyed out, "Đồng bộ" badge): automatically populated from `CanBoQuanLy.phanCong[]` if the user is linked to a CBQL via `CanBoQuanLy.taiKhoan`. Delete is disabled — to change them, edit the CBQL's `phanCong` in the Khóa học page. See [`docs/workflows/teacher-scope-sync.md`](../../../workflows/teacher-scope-sync.md).
   - **Manual rows** (below, editable): admin adds these directly. Each row has `khoa` + `daiDoi` dropdowns and two wildcard toggles (`allKhoa`, `allDaiDoi`). Save sends only these.
4. Save. The backend's `validateTeacherScopeAgainstDB` verifies every referenced khoa/daiDoi exists.

A teacher with only synced entries (no manual rows) passes validation — the popup accepts an empty manual list as long as the synced list is non-empty.

### Create a staff user scoped to one khoa

1. **Thêm người dùng**, fill credentials, `role = staff`.
2. Pick `allowedUnits.khoa` from the multi-select. Leave `allowAll` flags off.
3. Save.

### Disable an account

1. Open the user's row, edit, change `status` to `disabled`. Save.
2. The user can no longer log in; their existing access token also stops working at the next request (authGuard checks `status`).

### Reset a user's password

Admin edits the user, fills `password` (overwrites server-side via bcrypt). The user is **not** auto-logged-out — they keep their current access token until it expires.

### Delete a user

Confirmation modal. The route returns `400 CANNOT_DELETE_SELF` if you try to delete yourself, and `400 CANNOT_DELETE_LAST_ADMIN` if the user is the only remaining admin.

## Edge cases / gotchas

- **The page renders for staff/viewer too** (mounted resource), but write actions return `403 FORBIDDEN` because `/api/users/*` is admin-only inside the route file. Hide the action buttons or accept the 403; the UI may show actions speculatively.
- **A `teacher` user with an empty `teacherScope` ∪ `teacherScopeSynced` sees zero records** (sentinel filter `{ _id: { $in: [] } }` applied at query time). The UI no longer blocks saving in that state — admins may intentionally save a teacher with no assignments yet. Per-row validation (each manual row must specify Khóa+Đại đội or the matching "Tất cả" wildcard) still runs.
- **`allowedUnits` is staff-only.** Setting it on a viewer is allowed but the backend ignores it (`applyUnitScope` only runs for `role === 'staff'`).
- **Username is lowercased** automatically by Mongoose. `"Admin"` becomes `"admin"` in storage.
- **You cannot create the very first admin from this page** — `/api/auth/register` requires an existing admin to be logged in. Use `npm run create:admin` for bootstrap.
