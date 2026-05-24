# Kết quả khảo sát — Survey results

**Menu path**: Khảo sát chất lượng › Kết quả khảo sát

**Roles**: admin · staff · viewer (teacher: **hidden**; the resource is not mounted for teacher accounts and the menu item does not render)

**Source files**: [`frontend/src/resources/khaoSatChatLuong/ketQuaKhaoSat/ketQuaKhaoSat.tsx`](../../../../frontend/src/resources/khaoSatChatLuong/ketQuaKhaoSat/ketQuaKhaoSat.tsx) + `KetQuaKhaoSatView.tsx`

## When to use

Import Google Forms response exports and review per-tab processing statistics. The page is a full import-and-report workflow, not a redirector — all processing happens within the app.

## Layout

- Three survey-type tabs: `gvTuDanhGia`, `phanHoiGV`, `phanHoiTrungTam`.
- Each tab contains an **"Nhập phản hồi Google Forms"** file-upload button.
- Clicking the button opens a file-upload modal that sends `POST /api/khao-sat-chat-luong/process` with a `type` field (matching the active tab) and an optional `totalInstructors` field.
- After processing, statistics panels display per-tab results returned in the `X-KhaoSat-Stats` response header.

## Common tasks

- **Import a response export** — select the active tab, click **"Nhập phản hồi Google Forms"**, pick the file, submit.
- **Review statistics** — read the statistics panels rendered from the `X-KhaoSat-Stats` header after a successful upload.

## Edge cases / gotchas

- **Each tab is processed independently.** Switch to the correct tab before uploading; the `type` sent to the API matches the active tab.
- **No editing on this page.** The workflow is import-only; results are read-only once displayed.
