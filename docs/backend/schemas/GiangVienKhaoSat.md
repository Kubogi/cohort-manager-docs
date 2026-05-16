# GiangVienKhaoSat Schema

**Source**: [backend/src/models/GiangVienKhaoSat.js](../../../backend/src/models/GiangVienKhaoSat.js)

**Collection**: `giangVien` **(yes, camelCase — not `giang_vien_khao_sat`)** — primary cluster

**Last verified**: 2026-05-16

Per-year, per-instructor configuration for the "Phiếu lấy ý kiến phản hồi người học về giảng viên" feedback form. Each row links a `(nam, hoTen)` pair to a feedback URL that learners visit to rate that specific instructor.

This is **separate from** the operational `CanBoQuanLy` collection — the surveys may be conducted by instructors who aren't (or aren't yet) recorded as staff in the system. There is no foreign key between `GiangVienKhaoSat.hoTen` and `CanBoQuanLy.hoTen`; matching is by name string.

> **Naming gotcha.** The collection name on disk is the literal string `giangVien`, which collides visually with the *frontend resource name* (`giangVien` in App.tsx, which actually maps to `/api/can-bo-quan-ly`). They are different things. The frontend's "Giảng viên" page is unrelated to this collection.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `nam` | Number | Yes | — | Survey year (matches a row in [`KhaoSatChatLuongNam`](./KhaoSatChatLuongNam.md)). |
| `hoTen` | String | Yes | — | Instructor's full name, trimmed. Used as the natural key together with `nam`. |
| `linkPhanHoiNguoiHoc` | String | No | `''` | URL the learner is sent to in order to fill out the per-instructor feedback form. Trimmed. |
| `createdAt`, `updatedAt` | Date | Auto | — | Mongoose timestamps. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ nam: 1, hoTen: 1 }` | **unique** | One row per (year, instructor name). |
| `{ hoTen: 1 }` | non-unique | Filter "all years for instructor X". |

## Related

- [KhaoSatChatLuongNam](./KhaoSatChatLuongNam.md) — per-year center-level survey links
- [`/api/khao-sat-chat-luong/giang-vien`](../../api/README.md#quality-survey-apikhao-sat-chat-luong) — endpoints
- [Glossary: `quanLyLink`](../../glossary.md#operations-and-admin) — the admin-only UI that edits these
