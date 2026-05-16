# Link Management

**Triggered from**: [Quản lý link](../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md) (admin-only).

**Touches**: `/api/settings/training-schedule-link`, `/api/khao-sat-chat-luong/nam`, `/api/khao-sat-chat-luong/giang-vien`, `SystemSetting` + `KhaoSatChatLuongNam` + `GiangVienKhaoSat` models.

**Who can do this**: `admin`.

## Goal

Centralize every externally-hosted URL the SPA points at: the training-schedule iframe, the per-year survey URLs (center-level + per-instructor). Allow admin to update them without redeploying.

## Three groups of URLs

### Group 1 — Training schedule (single global)

- One URL stored as a `SystemSetting` row with `key: 'training-schedule-link'`.
- Read by [Lịch trình đào tạo](../frontend/pages/thong-tin-chung/lich-trinh-dao-tao.md) on page mount.

**Update flow**: admin pastes new URL → `PATCH /api/settings/training-schedule-link` → next page mount picks it up.

### Group 2 — Per-year survey links (one row per year)

- Stored in `KhaoSatChatLuongNam` with `nam` as the natural key.
- Two link fields per row: `linkGiangVienTuDanhGia`, `linkPhanHoiHoatDongTrungTam`.

**Update flow**:
1. Admin **Thêm năm** if the year doesn't exist (`POST /api/khao-sat-chat-luong/nam`).
2. Paste both URLs, save each (`PATCH /api/khao-sat-chat-luong/nam/:id`).

### Group 3 — Per-instructor feedback links (one row per (year, instructor))

- Stored in `GiangVienKhaoSat` with `(nam, hoTen)` as the natural key.
- Unique `(nam, hoTen)`; one URL per row.

**Update flow**:
1. Admin **Thêm giảng viên** in the chosen year (`POST /api/khao-sat-chat-luong/giang-vien`).
2. Edit the URL inline; save (`PATCH`).
3. Or **Xoá** (`DELETE`) to remove an entry.

## Side-effects

- Updates are immediately visible to all users on next page load — no caching layer to invalidate.
- Year config (`/nam`) has **no delete** by design — past years are retained for audit. To "deactivate" a year, blank both URLs.

## Failure modes

| Scenario | Result |
|---|---|
| Pastes a URL with no protocol | Frontend regex blocks the save. |
| Duplicate year on POST | `409` from the unique `nam` index. |
| Duplicate `(nam, hoTen)` on POST | `409` from the compound unique index. |

## Manual test recipe

- [ ] As admin, paste a new training-schedule URL. Reload [Lịch trình đào tạo](../frontend/pages/thong-tin-chung/lich-trinh-dao-tao.md). Confirm iframe loads the new URL.
- [ ] Add survey year 2026. Add two center URLs. Verify they appear on [Khảo sát sinh viên](../frontend/pages/khao-sat-chat-luong/khao-sat-sinh-vien.md) → 2026.
- [ ] Add an instructor "Test Teacher" with a URL. Verify they appear in the per-instructor table.
- [ ] Delete the test instructor. Re-add with the same name. Expect no `409`.
- [ ] As a non-admin user, attempt to load the Quản lý link page. Expect it not to render in the menu.
