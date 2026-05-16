# KhaoSatChatLuongNam Schema

**Source**: [backend/src/models/KhaoSatChatLuongNam.js](../../../backend/src/models/KhaoSatChatLuongNam.js)

**Collection**: `khao_sat_chat_luong_nam` (primary cluster)

**Last verified**: 2026-05-16

One row per **survey year** (e.g. 2024, 2025). Holds the two center-level links shown on the "Quản lý link" admin page: the instructor self-evaluation form and the learner-feedback-about-the-center form. Per-instructor feedback links live in the sibling [`GiangVienKhaoSat`](./GiangVienKhaoSat.md) collection.

## Fields

| Field | Type | Required | Unique | Default | Description |
|---|---|---|---|---|---|
| `nam` | Number | Yes | Yes | — | The survey year. Integer (e.g. `2025`). |
| `linkGiangVienTuDanhGia` | String | No | — | `''` | URL to the "Phiếu giảng viên tự đánh giá" form. Trimmed. |
| `linkPhanHoiHoatDongTrungTam` | String | No | — | `''` | URL to the "Phiếu phản hồi người học về hoạt động Trung tâm" form. Trimmed. |
| `createdAt`, `updatedAt` | Date | Auto | — | — | Mongoose timestamps. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ nam: 1 }` | **unique** | Exactly one row per year. |

## Related

- [GiangVienKhaoSat](./GiangVienKhaoSat.md) — per-instructor feedback URLs for each year
- [`/api/khao-sat-chat-luong/nam`](../../api/README.md#quality-survey-apikhao-sat-chat-luong) — endpoints
- [`/docs/workflows/quality-survey.md`](../../workflows/quality-survey.md) — end-to-end survey flow
