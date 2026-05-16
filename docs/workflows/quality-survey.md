# Quality Survey

**Triggered from**: [Quản lý link](../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md) (admin config), [Khảo sát chất lượng](../frontend/pages/khao-sat-chat-luong/) pages (distribution).

**Touches**: `/api/khao-sat-chat-luong/nam` (per-year center links), `/api/khao-sat-chat-luong/giang-vien` (per-instructor links), `KhaoSatChatLuongNam` + `GiangVienKhaoSat` models, plus the external survey hosting (Google Forms etc.) and the templates at `backend/forms/khao-sat-chat-luong/`.

**Who can do this**: `admin` for configuration; everyone with menu access for distribution.

## Goal

Run the annual quality survey: collect instructor self-evaluations + learner feedback (on the center and on each instructor). This system does not host the survey itself — it stores the **URLs** to externally-hosted forms and surfaces them to the audience.

## Phases

### Phase 1 — admin configures the year

1. Open [Quản lý link](../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md), Section 2.
2. Add a new year if needed (`POST /api/khao-sat-chat-luong/nam`).
3. Paste the two center-level URLs (`linkGiangVienTuDanhGia`, `linkPhanHoiHoatDongTrungTam`); save each via `PATCH /api/khao-sat-chat-luong/nam/:id`.
4. Section 3: add each instructor with `POST /api/khao-sat-chat-luong/giang-vien` (one row per `(nam, hoTen)`), paste their per-instructor feedback URL.

### Phase 2 — instructors fill self-evaluation

1. Each instructor opens [Giảng viên tự đánh giá](../frontend/pages/khao-sat-chat-luong/giang-vien-tu-danh-gia.md) for the current year.
2. Clicks the link, fills out the externally-hosted form.
3. The system does not record completion locally — that's tracked in the external form's response sheet.

### Phase 3 — students fill feedback

1. Operator opens [Khảo sát sinh viên](../frontend/pages/khao-sat-chat-luong/khao-sat-sinh-vien.md) for the current year.
2. For *Phản hồi về Trung tâm*, copies the URL and distributes to students (Zalo, posters).
3. For *Phản hồi về Giảng viên*, copies one URL per instructor and distributes accordingly. The students fill the externally-hosted forms.

### Phase 4 — review results

1. Open [Kết quả khảo sát](../frontend/pages/khao-sat-chat-luong/ket-qua-khao-sat.md).
2. The page links out to the external aggregation (Google Sheet etc.).
3. This system does not currently store, aggregate, or visualize responses — that's a future feature.

## Side-effects

- **None on this system's DB** beyond the URL config rows.
- The external survey provider holds all PII / response data.

## Failure modes

| Scenario | Result |
|---|---|
| Admin forgets to configure a year before survey time | Distribution pages show empty-state. Catch with a calendar reminder; the system has no automated nag. |
| Duplicate instructor name within a year | `409` on `POST /api/khao-sat-chat-luong/giang-vien`. Use a name suffix or remove the older row. |
| External form host goes down | Out of this system's control. Surface the failure in the UI as "Cannot reach the form" hint. |

## Manual test recipe

- [ ] Admin: create year `2026`, paste both center URLs, save.
- [ ] Admin: add a `GiangVien khaoSat` entry for "Trần Văn A" in 2026 with a Google Forms URL.
- [ ] Non-admin: open `Khảo sát sinh viên`, year 2026. Confirm both center URLs and the instructor URL appear and copy-paste works.
- [ ] Non-admin (teacher): open `Giảng viên tự đánh giá`, year 2026. Confirm the GV self-eval link appears.
- [ ] Admin: delete a typo'd instructor row, re-add with the correct name. Confirm no `409` on the re-add.
