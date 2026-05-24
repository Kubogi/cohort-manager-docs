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
- **Admin only**: a "Kết quả khảo sát" upload slot below the link panel. Holds one file per year (typically the exported response spreadsheet). Selecting a file uploads immediately; the saved file shows with `Tải về` / `Xóa` buttons. Stored as an `Attachment` with `ownerType: 'KhaoSatTuDanhGiaNam'`, keyed by the `KhaoSatChatLuongNam` year row. Hidden entirely for staff / viewer / teacher.

## Common tasks

- **Open the form to fill it out** — pick the year, click the link. Opens in a new tab.
- **Share the form URL** — click copy, paste into Zalo/email.
- **Archive the response export (admin)** — pick the year, choose the file in the upload slot; it auto-uploads. Replace by uploading again, delete via the `Xóa` button.

## Edge cases / gotchas

- **The URL is per-year.** Last year's URL doesn't auto-roll into this year. Admin configures it via [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md) when the new survey opens.
- **No data captured here.** The form responses go wherever the URL points (typically Google Forms → a Sheet). This page is purely a redirector — the uploaded result file is just an attached archive copy.
- **Upload is admin-only.** The slot is invisible to non-admins, and the backend rejects non-admin POST/DELETE on this `ownerType` (`403 FORBIDDEN`). Reads remain open so a future download-only surface for staff/viewer is a UI-only change.
