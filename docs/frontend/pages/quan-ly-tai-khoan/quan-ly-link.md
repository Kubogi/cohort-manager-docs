# Quản lý link — Link management

**Menu path**: Quản lý tài khoản › Quản lý link

**Roles**: admin only

**Source files**: [`frontend/src/resources/quanLyTaiKhoan/quanLyLink/quanLyLink.tsx`](../../../../frontend/src/resources/quanLyTaiKhoan/quanLyLink/quanLyLink.tsx) + companion folder

**Related API endpoints**:
- [`GET/PATCH /api/settings/training-schedule-link`](../../../api/README.md#system-settings-apisettings)
- [`/api/khao-sat-chat-luong/nam`](../../../api/README.md#quality-survey-apikhao-sat-chat-luong) — per-year survey URLs
- [`/api/khao-sat-chat-luong/giang-vien`](../../../api/README.md#quality-survey-apikhao-sat-chat-luong) — per-instructor feedback URLs

**Related workflows**: [`link-management.md`](../../../workflows/link-management.md), [`quality-survey.md`](../../../workflows/quality-survey.md)

## When to use

Admin-only configuration of every externally-hosted URL that the SPA embeds or links to. Three independent groups of links live here:

1. **Training schedule iframe** — a single URL shown on the "Lịch trình đào tạo" page.
2. **Per-year quality-survey forms** — two URLs per survey year (instructor self-evaluation + learner feedback on the center).
3. **Per-instructor feedback URLs** — one URL per `(year, instructor name)` pair, shown to learners when filling out the "feedback on instructor" form.

Despite the page having no tabbed UI, the three groups are visually distinct sections. Document them as three subsections rather than tabs.

## Layout

The page is a long form, top to bottom:

```
┌─ Section 1: Lịch trình đào tạo ──────────────────────────────┐
│ [single URL input]            [Lưu]                            │
└────────────────────────────────────────────────────────────────┘

┌─ Section 2: Khảo sát chất lượng — theo năm ─────────────────┐
│ [Year picker]   [+ Thêm năm]                                  │
│   ─ 2024:                                                     │
│       Phiếu Giảng viên tự đánh giá   [URL input]   [Lưu]      │
│       Phiếu phản hồi về Trung tâm    [URL input]   [Lưu]      │
│   ─ 2025: …                                                   │
└────────────────────────────────────────────────────────────────┘

┌─ Section 3: Khảo sát chất lượng — theo giảng viên ────────────┐
│ [Year filter]   [+ Thêm giảng viên]                            │
│ Table: { hoTen, linkPhanHoiNguoiHoc, edit, delete }           │
└────────────────────────────────────────────────────────────────┘
```

## Sections (logical "tabs")

### Section 1 — Training schedule link

**What it does:**
- `GET /api/settings/training-schedule-link` on mount, populates the input.
- **Lưu** sends `PATCH /api/settings/training-schedule-link` with the new value.

**Validation:**
- Must be `http://` or `https://` (frontend regex). Submitting other schemes is rejected without hitting the backend.

### Section 2 — Per-year survey links

**What it does:**
- `GET /api/khao-sat-chat-luong/nam` lists all configured years.
- **Thêm năm** opens a small dialog for the year integer. Calls `POST /api/khao-sat-chat-luong/nam` (only `nam` is required; the two link fields default to empty strings).
- Editing the two URL inputs and clicking **Lưu** for a row calls `PATCH /api/khao-sat-chat-luong/nam/:id` with the changed fields.
- There's no delete for a year (intentional — past years' configurations are kept for audit). To "deactivate" a year, blank both URLs.

### Section 3 — Per-instructor feedback links

**What it does:**
- `GET /api/khao-sat-chat-luong/giang-vien?nam=…` lists instructors for the chosen year.
- **Thêm giảng viên** opens a dialog with `nam` (preselected), `hoTen`, optional URL. Calls `POST /api/khao-sat-chat-luong/giang-vien`. Unique on `(nam, hoTen)` — adding a duplicate within a year returns 409.
- Inline edit of `hoTen` or `linkPhanHoiNguoiHoc` saves via `PATCH /api/khao-sat-chat-luong/giang-vien/:id`.
- **Xoá** calls `DELETE /api/khao-sat-chat-luong/giang-vien/:id`.

## Common tasks

### Update the training-schedule iframe to point at a new Google Sheets URL

1. Section 1, paste the new URL, click **Lưu**.
2. The "Lịch trình đào tạo" page picks it up on next render — no backend restart needed.

### Configure surveys for the 2026 academic year

1. Section 2, **Thêm năm**, enter `2026`. Save.
2. The newly-created row appears. Paste the two URLs, save each.
3. Section 3, change the year filter to `2026`, **Thêm giảng viên** for each instructor being evaluated. Paste their feedback URL.

### Remove a typo'd instructor entry

1. Section 3, **Xoá** the row. The unique `(nam, hoTen)` lets you re-add with the corrected name immediately.

## Edge cases / gotchas

- **Admin-only.** `App.tsx` only mounts the resource when `permissions === 'admin'`. Even an URL-typed access by a non-admin will land on a "not found" page.
- **Year is an integer, not a date.** `nam: 2025`, not `"2025"`. The schema's `Number` type enforces this server-side.
- **`hoTen` matching is purely by string.** This collection's instructor names are not linked to `CanBoQuanLy`. If you rename an instructor in `can_bo_quan_ly`, this collection is **not** updated.
- **Per-year config has no `delete`.** Trying to remove an entire year requires a manual mongo operation. Use blanked links as a soft-deactivate.
- **The training-schedule setting uses `systemsettings` (key/value) under the hood.** Future settings should follow the same pattern (one `key`, one route). See [SystemSetting schema](../../../backend/schemas/SystemSetting.md).
