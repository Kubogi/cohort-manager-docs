# Quản lý danh sách người dùng — User management

**Menu path**: Quản lý tài khoản › Quản lý danh sách người dùng

**Roles**: admin · staff · viewer *(not visible to teacher)* — most write actions are **admin-only** at the route level

**Source files**: [`frontend/src/resources/quanLyTaiKhoan/quanLyDanhSachNguoiDung/quanLyDanhSachNguoiDung.tsx`](../../../../frontend/src/resources/quanLyTaiKhoan/quanLyDanhSachNguoiDung/quanLyDanhSachNguoiDung.tsx) + companion folder

**Related API endpoints**: [`/api/users/*`](../../../api/README.md#users-apiusers), [`POST /api/auth/register`](../../../api/endpoints/auth/post-register.md)

**Related workflows**: [`user-management.md`](../../../workflows/user-management.md)

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
3. The modal expands to show the **TeacherScopeEditor** — a separate 820px modal that lets admin specify one or more `(khoa, daiDoi, allKhoa, allDaiDoi)` entries.
4. Each entry has dropdowns for `khoa` and `daiDoi` plus the two wildcard toggles. The editor enforces consistency: setting `allKhoa: true` hides the `khoa` dropdown for that entry.
5. Save. Backend `validateTeacherScopeAgainstDB` verifies every referenced khoa/daiDoi exists in the primary cluster.

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
- **The `teacher` role requires `teacherScope` to be meaningful.** A `teacher` user with an empty `teacherScope` sees zero records (sentinel filter). The TeacherScopeEditor flags this.
- **`allowedUnits` is staff-only.** Setting it on a viewer is allowed but the backend ignores it (`applyUnitScope` only runs for `role === 'staff'`).
- **Username is lowercased** automatically by Mongoose. `"Admin"` becomes `"admin"` in storage.
- **You cannot create the very first admin from this page** — `/api/auth/register` requires an existing admin to be logged in. Use `npm run create:admin` for bootstrap.
