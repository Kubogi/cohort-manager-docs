# Kết quả khảo sát — Survey results

**Menu path**: Khảo sát chất lượng › Kết quả khảo sát

**Roles**: admin · staff · viewer (teacher: **hidden**; the resource is not mounted for teacher accounts and the menu item does not render). Archive write (upload via `nam`) is **admin-only**.

**Source files**: [`frontend/src/resources/khaoSatChatLuong/ketQuaKhaoSat/ketQuaKhaoSat.tsx`](../../../../frontend/src/resources/khaoSatChatLuong/ketQuaKhaoSat/ketQuaKhaoSat.tsx) + `KetQuaKhaoSatView.tsx` + `useKetQuaKhaoSatData.ts`

## When to use

Import Google Forms response exports, review per-tab processing statistics, and archive both the uploaded source file and the generated report against the chosen năm. All processing happens within the app; the generated report streams back to the browser for immediate download AND is persisted on the server for re-download later.

## Layout

- **Năm khảo sát** picker at the top of the page. Required before any upload — gating the "Nhập phản hồi" button.
- Three survey-type tabs:
  - `gvTuDanhGia` — Giảng viên tự đánh giá
  - `phanHoiGV` — Phản hồi người học về giảng viên (**single merged sheet per năm** — not per teacher)
  - `phanHoiTrungTam` — Phản hồi người học về các hoạt động của Trung tâm
- **"Nhập phản hồi Google Forms"** button. Disabled until a năm is picked.
- Modal collects the file + (for `gvTuDanhGia` only) optional `totalInstructors`.
- After processing: the report Excel auto-downloads (existing UX) and the stats panel renders from the `X-KhaoSat-Stats` response header.
- Below the stats panel, an **archive box** for the active (tab, năm) shows two read-only rows:
  - "Tệp gốc đã nhập" — the uploaded source file
  - "Báo cáo đã tạo" — the generated report
  Both have `Tải về` / `Xóa` buttons. Empty rows show "Chưa có tệp.".

## Backend write surface

The upload routes through [`POST /api/khao-sat-chat-luong/process`](../../../api/endpoints/khao-sat-chat-luong/README.md#post-apikhao-sat-chat-luongprocess) with `nam` in the FormData. The endpoint runs the existing pipeline AND persists two Attachment rows:

| Tab | Source ownerType | Report ownerType |
|---|---|---|
| `gvTuDanhGia` | `KhaoSatGvTuDanhGiaSource` | `KhaoSatGvTuDanhGiaReport` |
| `phanHoiGV` | `KhaoSatPhanHoiGVSource` | `KhaoSatPhanHoiGVReport` |
| `phanHoiTrungTam` | `KhaoSatPhanHoiTrungTamSource` | `KhaoSatPhanHoiTrungTamReport` |

Re-uploading for the same `(năm, tab)` replaces both rows and unlinks the old disk files.

## Common tasks

- **Import a response export** — pick năm, select the active tab, click **Nhập phản hồi Google Forms**, pick the file, submit. The report downloads and both files appear in the archive box below the stats panel.
- **Review past statistics** — change năm + tab; the archive rows populate from the existing attachments. Re-running stats requires a fresh upload (stats aren't persisted).
- **Re-download an old report** — click `Tải về` on "Báo cáo đã tạo" in the archive box.
- **Delete a stored file** — `Xóa`. Admin-only.

## Edge cases / gotchas

- **Each tab is processed independently.** Switch to the correct tab before uploading; `type` sent to the API matches the active tab.
- **Re-upload replaces.** Two files per (năm, tab), one row each. A new upload for the same (năm, tab) clobbers both stored files.
- **`phanHoiGV` is per-năm, not per-teacher.** Earlier UI shipped one slot per (năm, teacher); the upload now expects one merged sheet for the whole năm (which the processor splits into per-teacher report sheets).
- **`nam` is required for the archive.** Omitting it (e.g. a direct cURL call) makes the endpoint behave as it did before the archive migration — staff-allowed, no persistence.
- **The năm value silently switches the output layout for 2026.** The backend dispatches on `nam === 2026` and produces structurally different reports for Tabs 1 & 2 (Tab 3 unchanged). See [the API doc](../../../api/endpoints/khao-sat-chat-luong/README.md#per-năm-pipeline-branches) for the per-tab v2024 vs v2026 layout table.
  - **Tab 1 `gvTuDanhGia` (năm 2026)** — source now needs a `4. Khoa` column at position E (e.g. `Khoa Quân Sự` / `Khoa Chính trị`). Output is 3 sheets (`2025-2026` main + per-khoa matrices) with 29 questions. Rows whose Khoa value isn't recognized land in `stats.warnings` but don't fail the upload.
  - **Tab 2 `phanHoiGV` (năm 2026)** — each per-teacher sheet in the upload must carry an explicit ARGB tab color (`FF002060` navy = Quân sự, `FF00B050` green = Chính trị, `FFFF0000` red = exclude). Set via right-click tab → Tab Color → **More Colors → Custom** (Excel's "Theme Colors" picker is NOT recognized). Uncolored sheets cause a `400 UNBUCKETED_TEACHER_SHEETS` with the offending names listed.
  - **Tab 3 `phanHoiTrungTam` (năm 2026)** — no change, reuses v2024 pipeline.