# Tra cứu — Certificate / grade lookup

**Menu path**: Quản lý điểm › Tra cứu

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/quanLyDiem/traCuu.tsx`](../../../../frontend/src/resources/quanLyDiem/traCuu.tsx) + companion folder

**Related API endpoints**: [`POST /api/tra-cuu/grades`](../../../api/endpoints/tra-cuu/README.md), [`POST /api/tra-cuu/certificates`](../../../api/endpoints/tra-cuu/README.md)

## When to use

Look up a student's grades and/or certificate by partial identity — name + date of birth, or maSV + sending school. Useful when the student doesn't know their `maSV` from this center but you know their identity from the partner school.

## Layout

- A search form at the top with two equivalent paths:
  - Path A: `họ tên` + `ngày sinh`.
  - Path B: `maSV` + `đơn vị liên kết` (partner school).
- Two result tables rendered sequentially in a **vertical flex column** — there is no side-by-side layout at any breakpoint:
  - **Bảng điểm** (`GradeResultTable`) — grades for the matched student (from `SinhVien.diem[]` in the primary cluster).
  - **Chứng chỉ** (`CertificateResultTable`) — certificate record from the secondary cluster's `students` collection.
- Two buttons under the form: **Tra cứu** submits both queries; **Xóa** clears every input field, both result tables, the error banner, and resets `hasSearched`.

## Common tasks

- **Look up by name + DOB** — Path A. The backend tries an exact match; case-insensitive on `hoTen`.
- **Look up by maSV** — Path B. Useful when the partner school knows their student's number but not the matching record here.
- **Both tables show data** — student has both an operational record here and a historical certificate.
- **Only the Chứng chỉ table shows data** — graduated student; current record may have been archived or never existed in `sinh_vien`.
- **Only the Bảng điểm table shows data** — student is currently enrolled; certificate is not yet issued.

## Edge cases / gotchas

- **Two clusters, two queries.** The page makes two parallel requests; if either fails, the other still shows. Don't take a missing table as proof the student doesn't exist — check the other result.
- **`ngay_sinh` in the secondary is a *string*.** The lookup uses string comparison, so date format matters: send `dd/mm/yyyy`, not ISO. The frontend handles the conversion.
- **`donViLienKet` in Path B is the secondary-cluster school document.** Use the SearchableSelect — typing the school name fuzzy-matches.
- **No edits here.** Read-only on both tables.
- **No pagination.** Each search fetches the full result set in one round-trip with `limit=500` — the cap enforced by the backend Joi validator (`traCuu.validator.js`). Identity-keyed lookups (maSV+truong or hoTen+ngaySinh) realistically return 1–10 rows, so 500 is far above the worst-case sibling-name collision. Sending a `limit` above 500 produces `"limit" must be less than or equal to 500` (400) — to lift the cap, both the validator and `FULL_LIMIT` in `useTraCuuData` must change in lockstep.
