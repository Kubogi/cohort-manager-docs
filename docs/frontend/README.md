# Frontend Page Reference

User-facing documentation for every page in the SPA. Use this when you need to understand what a page does, what tabs it has, or which API endpoints it calls — without reading the source.

The page tree mirrors the Vietnamese menu in `frontend/src/App.tsx`. The menu has six top-level sections plus one external link. Each section appears in this index in the same order it appears in the UI.

For the technical scaffolding (react-admin shell, dataProvider, design tokens) see [`/docs/architecture/frontend.md`](../architecture/frontend.md). For domain term translations see [`/docs/glossary.md`](../glossary.md).

## Menu hierarchy

### Quản lý tài khoản

Account management. Visible to all roles; teacher sees only `caiDatTaiKhoan`.

| Page | Roles | Doc |
|---|---|---|
| Cài đặt tài khoản | all | [pages/quan-ly-tai-khoan/cai-dat-tai-khoan.md](pages/quan-ly-tai-khoan/cai-dat-tai-khoan.md) |
| Quản lý danh sách người dùng | admin, staff, viewer | [pages/quan-ly-tai-khoan/quan-ly-danh-sach-nguoi-dung.md](pages/quan-ly-tai-khoan/quan-ly-danh-sach-nguoi-dung.md) |
| Quản lý link | admin only | [pages/quan-ly-tai-khoan/quan-ly-link.md](pages/quan-ly-tai-khoan/quan-ly-link.md) |

### Thông tin chung

General information. Hidden from teacher.

| Page | Tabs | Doc |
|---|---|---|
| Khóa học | 2 | [pages/thong-tin-chung/khoa-hoc.md](pages/thong-tin-chung/khoa-hoc.md) |
| Giảng viên | — | [pages/thong-tin-chung/giang-vien.md](pages/thong-tin-chung/giang-vien.md) |
| Lịch trình đào tạo | — | [pages/thong-tin-chung/lich-trinh-dao-tao.md](pages/thong-tin-chung/lich-trinh-dao-tao.md) |

### Quản lý sinh viên

Student management. Hidden from teacher.

| Page | Tabs | Doc |
|---|---|---|
| Cơ sở dữ liệu sinh viên | — | [pages/quan-ly-sinh-vien/co-so-du-lieu-sinh-vien.md](pages/quan-ly-sinh-vien/co-so-du-lieu-sinh-vien.md) |
| Biên chế đại đội tự động | 2 | [pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md](pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md) |
| Quản lý các QĐ/CV | — | [pages/quan-ly-sinh-vien/quan-ly-cac-quyet-dinh.md](pages/quan-ly-sinh-vien/quan-ly-cac-quyet-dinh.md) |
| Biểu mẫu & in ấn | — | [pages/quan-ly-sinh-vien/bieu-mau-in-an.md](pages/quan-ly-sinh-vien/bieu-mau-in-an.md) |
| Hồ sơ sức khỏe | 3 | [pages/quan-ly-sinh-vien/ho-so-suc-khoe.md](pages/quan-ly-sinh-vien/ho-so-suc-khoe.md) |
| Báo cáo | — | [pages/quan-ly-sinh-vien/bao-cao.md](pages/quan-ly-sinh-vien/bao-cao.md) |

### Quản lý giảng dạy

Teaching management. Visible to all including teacher.

| Page | Tabs | Doc |
|---|---|---|
| Lịch huấn luyện | — | [pages/quan-ly-giang-day/lich-huan-luyen.md](pages/quan-ly-giang-day/lich-huan-luyen.md) — **stub page** |
| Sổ tay giảng viên | 5 | [pages/quan-ly-giang-day/so-tay-giang-vien.md](pages/quan-ly-giang-day/so-tay-giang-vien.md) |
| Kho học liệu số | — | [pages/quan-ly-giang-day/kho-hoc-lieu-so.md](pages/quan-ly-giang-day/kho-hoc-lieu-so.md) |

### Quản lý điểm

Grade management. Visible to all including teacher.

| Page | Tabs | Doc |
|---|---|---|
| Nhập điểm | 2 | [pages/quan-ly-diem/nhap-diem.md](pages/quan-ly-diem/nhap-diem.md) |
| Tra cứu | — | [pages/quan-ly-diem/tra-cuu.md](pages/quan-ly-diem/tra-cuu.md) |
| Thống kê, báo cáo | — | [pages/quan-ly-diem/thong-ke-bao-cao.md](pages/quan-ly-diem/thong-ke-bao-cao.md) |

### Khảo sát chất lượng

Quality survey. Visible to all including teacher.

| Page | Doc |
|---|---|
| Giảng viên tự đánh giá | [pages/khao-sat-chat-luong/giang-vien-tu-danh-gia.md](pages/khao-sat-chat-luong/giang-vien-tu-danh-gia.md) |
| Khảo sát sinh viên | [pages/khao-sat-chat-luong/khao-sat-sinh-vien.md](pages/khao-sat-chat-luong/khao-sat-sinh-vien.md) |
| Kết quả khảo sát | [pages/khao-sat-chat-luong/ket-qua-khao-sat.md](pages/khao-sat-chat-luong/ket-qua-khao-sat.md) |

### CSDL Chứng chỉ

External link, not a SPA page. Clicking opens the [Certificate Management](https://github.com/Kubogi/certificate-management-docs) system in a new tab. Wired in `App.tsx` via `window.open`.

## Role-visibility matrix

| Section | admin | staff | viewer | teacher |
|---|:-:|:-:|:-:|:-:|
| Quản lý tài khoản → Cài đặt tài khoản | ✓ | ✓ | ✓ | ✓ |
| Quản lý tài khoản → Quản lý danh sách người dùng | ✓ | ✓ | ✓ | — |
| Quản lý tài khoản → Quản lý link | ✓ | — | — | — |
| Thông tin chung | ✓ | ✓ | ✓ | — |
| Quản lý sinh viên | ✓ | ✓ | ✓ | — |
| Quản lý giảng dạy | ✓ | ✓ | ✓ | ✓ |
| Quản lý điểm | ✓ | ✓ | ✓ | ✓ |
| Khảo sát chất lượng | ✓ | ✓ | ✓ | ✓ |

For role + per-unit scoping details see [`/docs/architecture/auth.md`](../architecture/auth.md).

## Per-page doc template

Every file under `pages/` follows the same shape. If you write a new one, copy this skeleton:

```markdown
# <Vietnamese label> — <one-line English purpose>

**Menu path**: Quản lý X › <label>
**Roles**: admin / staff / viewer / teacher (which can see it)
**Source files**: <entry .tsx + companion folder>
**Related API endpoints**: <links to docs/api/endpoints/...>
**Related workflows**: <links to docs/workflows/...>

## When to use
2–3 sentences.

## Layout
ASCII or bullet description of the visible regions.

## Tabs   ← only if the page has tabs
Per tab: name, what it does, what it shows, what actions it offers.

## Common tasks
Step-by-step recipes for 3–5 user goals.

## Edge cases / gotchas
Behaviors that surprise users.
```

## Status of the per-page docs

The page docs are written incrementally. The largest and most-used pages are documented first; smaller stub pages may be one short paragraph until they grow.
