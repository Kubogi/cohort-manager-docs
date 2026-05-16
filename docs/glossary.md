# Glossary (EN ↔ VI)

This is the canonical translation reference for the project. Vietnamese domain terms appear throughout the codebase (resource names, route paths, model fields) and in the UI; this file maps them to English so non-Vietnamese-speaking developers can read the code and docs.

When you write new documentation, prefer the English term in prose and put the Vietnamese term in parentheses on first use: e.g. *"Each course (khóa) has one or more battalions (đại đội)."*

## Organizational structure

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Khóa | Course / cohort / intake | `khoa`, `khoaHoc` | A class year/intake. Top-level grouping of students. The endpoint is `/api/khoa-hoc`; the unit collection is `khoa`. |
| Đại đội | Battalion / company | `daiDoi` | A military training unit. Students are assigned to a đại đội within a khóa. |
| Đơn vị liên kết | Partner organization / linked school | `donViLienKet` | An external school (đại học, cao đẳng) that sends students to the center. Lives in the **secondary** Mongo connection. |
| Trường | School (alias of đơn vị liên kết) | `truong` | The field name on `SinhVien` / `HoSoSucKhoe` referencing the partner school. |
| Đơn vị | Unit (umbrella term) | `donVi` | Umbrella for khoa + đại đội + đơn vị liên kết. The route prefix `/api/don-vi` covers all three. |
| Hệ | Education system / track | `he` | Either `"Đại học"` (university) or `"Cao đẳng"` (college). Drives which Excel template fires in `bieu-mau` export. |
| Hệ ĐH / Hệ CĐ | Đại học / Cao đẳng | — | University / College, respectively. Folder names in `backend/forms/`. |
| Khung | Frame / track | `khung` | Either `"Chính trị"` or `"Quân sự"` on `HoSoSucKhoe`. |

## People

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Sinh viên | Student | `sinhVien`, `coSoDuLieuSinhVien` | Core domain entity. Embedded `diem` array holds grades. |
| Giảng viên | Instructor / teacher | `giangVien` | UI label and resource name. The **backend model** is `CanBoQuanLy` (`/api/can-bo-quan-ly`). |
| Cán bộ quản lý | Management staff | `canBoQuanLy` | Backend name for instructor / staff records. The frontend resource `giangVien` maps to this endpoint via `dataProvider.resourcePathMap`. |
| Học viên | Trainee / learner | — | Used interchangeably with sinh viên in some UI labels. |
| Người dùng | User / account | `nguoiDung`, `users` | Login account, distinct from the person record. |

## Records and decisions

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Quyết định / QĐ | Administrative decision | `quyetDinh`, `quanLyCacQuyetDinh` | A formal decision affecting a student (Hoãn học, Thôi học, Đình chỉ, Miễn học). Has a status machine — see [decision-processing workflow](workflows/decision-processing.md). |
| CV (công văn) | Official letter | — | Used in the UI label "QĐ/CV" but the data model only persists quyết định. |
| Loại QĐ | Decision type | `loaiQD` | Free-text label like "Hoãn học", "Đình chỉ học tập 1 năm", etc. |
| Số QĐ | Decision number | `soQD` | Document number printed on the physical decision. Unique per student. |
| Hồ sơ sức khỏe | Health record | `hoSoSucKhoe` | One record per hospital visit / health event. Status `Bình thường` or `Viện`. |
| Bệnh viện | Hospital | `benhVien` | Free-text hospital name on health record. |
| Trạng thái | Status | `trangThai` | Polymorphic. On `SinhVien`: `Đang học / Hoãn học / Thôi học / Đình chỉ / Miễn học`. On `QuyetDinh`: `Chưa xử lý / Đã xử lý`. On `HoSoSucKhoe`: `Bình thường / Viện`. |
| Đang học / Hoãn học / Thôi học / Đình chỉ / Miễn học | Studying / Suspended / Withdrawn / Expelled / Exempt | (sinhVien.trangThai enum) | The five terminal states a student can be in. |
| Chưa xử lý / Đã xử lý | Unprocessed / Processed | (quyetDinh.trangThai enum) | Whether a quyết định has been applied to the student. |

## Grades and assessment

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Điểm | Grade / score | `diem` | Embedded subdocument inside `SinhVien.diem[]`. Endpoint `/api/diem`. |
| Môn | Subject | `mon` | One of 7 fixed military-training subjects (see `MON_ENUM` in `backend/src/models/sinhVien.js`). |
| Điểm thường xuyên | Continuous-assessment grade | `thuongXuyen` | Weight: 1 on Đại học, 1 on Cao đẳng. |
| Điểm miệng | Oral grade | `mieng` | Weight: 0 on Đại học (not used), 1 on Cao đẳng. |
| Điểm giữa học phần / giữa HP | Midterm grade | `giuaHP` | Weight: 3 on Đại học (first 4 subjects), 2 on Cao đẳng. |
| Điểm hết học phần / hết HP | Final grade | `hetHP` | Weight: 6 on both systems. |
| TBM / điểm trung bình môn | Subject average | `tbMon` | Auto-computed by the `pre('validate')` hook in `diemSchema`. Read-only from the API perspective. |
| Tổng kết cuối khóa | End-of-course summary | — | A second tab in the "Nhập điểm" page. |
| Xếp loại | Classification / rank | `xepLoai` | Field on the external certificate-lookup record (e.g. Giỏi, Khá). |

## Files and templates

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Biểu mẫu | Form / template | `bieuMau` | Excel template for printing rosters, grade books, etc. Endpoint `/api/bieu-mau/export`. |
| In ấn | Printing | — | "Biểu mẫu & in ấn" is the page name (Forms & Printing). |
| Học liệu | Learning material | `hocLieu` | File/folder in the digital materials repository. |
| Kho học liệu số | Digital learning materials repository | `khoHocLieuSo` | The page that browses học liệu. |
| Giáo án | Teaching plan | `giaoAn` | One of the three `hoc-lieu` drives. |
| Giáo trình | Textbook / coursebook | `giaoTrinh` | One of the three drives. |
| Tài liệu tham khảo | Reference material | `taiLieuThamKhao` | One of the three drives. |
| Sổ điểm | Grade book | `soDiem` | A type of biểu mẫu — multi-sheet Excel output. |
| Sổ tay giảng viên | Instructor handbook | `soTayGiangVien` | The instructor's multi-tab dashboard page. |
| Danh sách | Roster / list | `danhSach` | Used in `(he, danhSach, mon)` triple to select an Excel template. |

## Operations and admin

| Vietnamese | English | Code identifier(s) | Notes |
|---|---|---|---|
| Quản lý đào tạo | Training management | (app title in `App.tsx`) | The top-bar label of the SPA. |
| Biên chế đại đội tự động | Automatic battalion assignment | `bienCheDaiDoiTuDong` | Rule-based bulk assignment of students to units. |
| Chuyển đại đội | Battalion transfer | `chuyenDoi` | Bulk move students between units. |
| Tra cứu | Lookup / search | `traCuu` | Public-facing certificate lookup endpoint `/api/tra-cuu`. |
| Báo cáo | Report | `baoCao` | Aggregated student report. |
| Thống kê | Statistics | `thongKe` | Statistical view (often paired with báo cáo). |
| Khảo sát chất lượng | Quality survey | `khaoSatChatLuong` | Annual survey of instructors and students. |
| Giảng viên tự đánh giá | Instructor self-evaluation | `giangVienTuDanhGia` | Survey tab. |
| Phiếu phản hồi | Feedback form | — | UI label inside the survey link manager. |
| Kết quả khảo sát | Survey results | `ketQuaKhaoSat` | Aggregated survey output. |
| Trực quân sự | Military duty / duty roster | `trucQuanSu` | Daily-roster collection. Endpoint `/api/truc-quan-su`. |
| Lịch huấn luyện | Training schedule | `lichHuanLuyen` | Currently a stub page. |
| Lịch trình đào tạo | Training timetable | `lichTrinhDaoTao` | Embeds an external link configured via `/api/settings/training-schedule-link`. |
| Cài đặt tài khoản | Account settings | `caiDatTaiKhoan` | Per-user settings (change password). |
| Quản lý link | Link management | `quanLyLink` | Admin-only page for editing survey and training-schedule URLs. |

## Personal / record fields

Common Vietnamese field names that appear across `SinhVien`, `CanBoQuanLy`, etc.:

| Field | English |
|---|---|
| `maSV` | Student ID code |
| `cccd` | National ID number (Căn cước công dân) |
| `hoTen` | Full name |
| `ngaySinh` | Date of birth |
| `noiSinh` | Place of birth |
| `gioiTinh` | Gender (boolean: true = male, false = female by convention in this codebase) |
| `soDienThoai` | Phone number |
| `lop` | Class |
| `nganh` | Major / field of study |
| `ngayNhapHoc` | Enrollment date |
| `ngayVe` | Departure date |
| `ngayKiQD` | Decision-signing date |
| `lyDo` | Reason |
| `ghiChu` | Notes / remark |
| `capBac` | Military rank |
| `chucVu` | Job title / position |
| `donViQL` | Managing unit (free-text) |
| `taiKhoan` | Linked user account (FK to `User`) |
| `hieuLuc` | Validity period |

## Permissions and roles

| Vietnamese | English | Code identifier | Notes |
|---|---|---|---|
| `admin` | Administrator | — | Full access. Only role that can create users, manage links, and reach admin-only resources. |
| `staff` | Staff | — | Read/write access to most operational data. Subject to `allowedUnits` scoping. |
| `viewer` | Viewer | — | Read-only access. Subject to `allowedUnits` scoping. |
| `teacher` | Teacher | — | Restricted to a subset of pages (see `App.tsx` `teacherOnly` set). Subject to `teacherScope` per-unit filtering. |
| `allowedUnits` | Per-unit access scope | — | On `User`: lists of khoa/đại đội/đơn vị liên kết IDs (with `allowAll` overrides) that staff/viewer can access. |
| `teacherScope` | Teacher per-unit filter | — | On `User` (teacher role only): array of `{ khoa, daiDoi, allKhoa, allDaiDoi }` entries embedded in the JWT access token. |
