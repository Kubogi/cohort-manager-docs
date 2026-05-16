# Lịch trình đào tạo — Training schedule

**Menu path**: Thông tin chung › Lịch trình đào tạo

**Roles**: admin · staff · viewer *(not visible to teacher)*

**Source files**: [`frontend/src/resources/thongTinChung/lichTrinhDaoTao.tsx`](../../../../frontend/src/resources/thongTinChung/lichTrinhDaoTao.tsx)

**Related API endpoints**: [`GET /api/settings/training-schedule-link`](../../../api/README.md#system-settings-apisettings)

## When to use

Displays the training schedule from an externally-hosted page (typically Google Sheets or a partner-school calendar) inside an iframe. Read-only for everyone; the URL is configured by admin on [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md).

## Layout

The page is essentially a full-bleed iframe. There's a small header with the page label and a "Loading..." state until the iframe responds. If no link is configured, the page shows a friendly message instead of a broken iframe.

## Common tasks

- **View the schedule** — open the page.
- **Update the URL** — admin only, on [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md). Changes apply immediately on next page load; no backend restart.

## Edge cases / gotchas

- **Iframe restrictions.** Some external pages set `X-Frame-Options: DENY` or `Content-Security-Policy: frame-ancestors 'self'` — those won't render inside the iframe. The fix is on the host of the embedded content, not in this app.
- **No fallback URL.** If the configured URL is unreachable, the iframe shows the browser's "can't connect" page; the SPA doesn't intercept.
- **HTTPS only.** Embedding `http://` content on an `https://` deployment triggers mixed-content blocking. Use HTTPS URLs.
