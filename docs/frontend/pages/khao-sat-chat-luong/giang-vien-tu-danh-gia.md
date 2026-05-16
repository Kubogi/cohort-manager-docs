# Giảng viên tự đánh giá — Instructor self-evaluation

**Menu path**: Khảo sát chất lượng › Giảng viên tự đánh giá

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/khaoSatChatLuong/giangVienTuDanhGia/giangVienTuDanhGia.tsx`](../../../../frontend/src/resources/khaoSatChatLuong/giangVienTuDanhGia/giangVienTuDanhGia.tsx) + `GiangVienTuDanhGiaView.tsx`

**Related API endpoints**: [`/api/khao-sat-chat-luong/nam`](../../../api/README.md#quality-survey-apikhao-sat-chat-luong)

## When to use

Open or distribute the annual instructor self-evaluation form. The form itself is hosted externally (Google Forms or similar); this page just surfaces the configured URL for the chosen year and provides a quick-copy / quick-open button.

## Layout

- Year picker (defaults to current).
- A panel showing the configured `linkGiangVienTuDanhGia` for that year as a clickable link + copy-to-clipboard button.
- If no URL is configured for the year, a friendly empty state with a hint to admin to set it on [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md).

## Common tasks

- **Open the form to fill it out** — pick the year, click the link. Opens in a new tab.
- **Share the form URL** — click copy, paste into Zalo/email.

## Edge cases / gotchas

- **The URL is per-year.** Last year's URL doesn't auto-roll into this year. Admin configures it via [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md) when the new survey opens.
- **No data captured here.** The responses go wherever the URL points (typically Google Forms → a Sheet). This page is purely a redirector.
