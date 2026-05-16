# Cài đặt tài khoản — Account settings

**Menu path**: Quản lý tài khoản › Cài đặt tài khoản

**Roles**: all (admin, staff, viewer, teacher)

**Source files**: [`frontend/src/resources/quanLyTaiKhoan/caiDatTaiKhoan/caiDatTaiKhoan.tsx`](../../../../frontend/src/resources/quanLyTaiKhoan/caiDatTaiKhoan/caiDatTaiKhoan.tsx) + companion folder

**Related API endpoints**: [`PATCH /api/auth/change-password`](../../../api/endpoints/auth/patch-change-password.md)

## When to use

Per-user account self-service. Today the only action is **change password**. Future: avatar, display name, notification preferences.

## Layout

- A single form: `mật khẩu cũ`, `mật khẩu mới`, `xác nhận mật khẩu mới`.
- A **Đổi mật khẩu** button.

## Common tasks

- **Change my password** — fill the three fields. The Joi schema enforces `oldPassword` non-empty, `newPassword ≥ 6 chars`, and that they differ.
- **Confirm field mismatch** is a frontend-only check before the network call.

## Edge cases / gotchas

- **Teachers cannot change their password via this page.** The backend route `PATCH /api/auth/change-password` does not authorize the `teacher` role — submitting returns `403 FORBIDDEN`. This is likely a backend bug (see [patch-change-password.md](../../../api/endpoints/auth/patch-change-password.md)).
- **Wrong old password** → `401 INVALID_OLD_PASSWORD`. Form shows the Vietnamese error.
- **New = old** → `400 SAME_PASSWORD`.
- **Password change does NOT invalidate existing tokens.** The user remains logged in with their current access token until it expires (1 h default). If you suspect the account is compromised, also have the admin disable + re-enable the user, or rotate `JWT_SECRET` server-wide.
