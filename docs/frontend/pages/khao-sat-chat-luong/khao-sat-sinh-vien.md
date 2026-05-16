# Khảo sát sinh viên — Student survey

**Menu path**: Khảo sát chất lượng › Khảo sát sinh viên

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/khaoSatChatLuong/khaoSatSinhVien/khaoSatSinhVien.tsx`](../../../../frontend/src/resources/khaoSatChatLuong/khaoSatSinhVien/khaoSatSinhVien.tsx) + `KhaoSatSinhVienView.tsx`

**Related API endpoints**: [`/api/khao-sat-chat-luong/nam`](../../../api/README.md#quality-survey-apikhao-sat-chat-luong), [`/api/khao-sat-chat-luong/giang-vien`](../../../api/README.md#quality-survey-apikhao-sat-chat-luong)

## When to use

Surface the student-facing feedback URLs for the chosen year. Two flavors:

- **Phản hồi về Trung tâm** — center-level feedback (one URL per year).
- **Phản hồi về Giảng viên** — per-instructor feedback (one URL per instructor per year).

Operators copy these URLs and distribute them to students via Zalo, posters, or in-class announcements.

## Layout

- Year picker.
- Section A: **Phản hồi về Trung tâm** — single link panel using `KhaoSatChatLuongNam.linkPhanHoiHoatDongTrungTam`.
- Section B: **Phản hồi về Giảng viên** — a `SearchableSelect` dropdown that lets the user select ONE instructor; after selecting, a single link row displays that instructor's `linkPhanHoiNguoiHoc` URL with a copy button. There is no multi-row table.

## Common tasks

- **Print or share a QR code for the center feedback URL** — copy the URL, paste into a QR generator, print.
- **Find an instructor's individual feedback link** — select the instructor from the dropdown, then click copy. Users select from a dropdown — they do not type into a free-text search field.

## Edge cases / gotchas

- **The instructor list here is *not* the same as `CanBoQuanLy`.** It's the `giangVien` collection (yes, name collision — see [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md)). The names are independent strings; renaming an instructor in `can_bo_quan_ly` does not propagate.
- **Empty URL panels** — if admin hasn't configured the URL for this year, the panel shows an empty state. Direct the operator to admin to fill it in.
- **Read-only on this page.** Add/remove instructors and edit URLs on [Quản lý link](../quan-ly-tai-khoan/quan-ly-link.md) (admin only).
