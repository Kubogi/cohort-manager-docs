# Grade Entry & Summary

**Triggered from**: [Nhập điểm](../frontend/pages/quan-ly-diem/nhap-diem.md) page (both tabs).

**Touches**: `POST/PATCH /api/diem`, embedded `SinhVien.diem[]`, `GET /api/diem/summary`, `POST /api/diem/import`, `GET /api/bieu-mau/export`.

**Who can do this**: `admin`, `staff`, `teacher` (writes; teacher restricted to in-scope students); `viewer` reads.

## Goal

Capture per-subject scores (thường xuyên, miệng, giữa HP, hết HP) for every student, auto-compute `tbMon` consistently, and produce both a per-cohort summary view and printable grade-book exports.

## The data model

Grades are **embedded** on `SinhVien.diem[]` — there is no separate `diem` collection. Each entry has:

```ts
{
  mon: <one of 7 MON_ENUM values>,
  thuongXuyen: 0..10,
  mieng?: 0..10,          // Cao đẳng only
  giuaHP: 0..10,
  hetHP: 0..10,
  tbMon: number,          // auto-computed by pre('validate')
  createdAt, updatedAt
}
```

The `pre('validate')` hook in [`backend/src/models/sinhVien.js`](../../backend/src/models/sinhVien.js#L42) recomputes `tbMon` on every save:

- For the first 4 subjects (Đại học track):
  `tbMon = (thuongXuyen × 1 + giuaHP × 3 + hetHP × 6) / 10`
- For the remaining 3 subjects (Cao đẳng track):
  `tbMon = (thuongXuyen × 1 + mieng × 1 + giuaHP × 2 + hetHP × 6) / 10`

Computation is done in integer arithmetic to avoid IEEE-754 drift — every `tbMon` is exactly representable as a multiple of 0.1.

## Tab 1 — Nhập điểm

For one selected `(khoa, daiDoi, mon)` triple, the page renders a table of students with one row per student and editable columns for the relevant grade components. Each `(student, mon)` cell maps to one entry in `student.diem[]`.

### Happy path

1. User selects `khoa`, `daiDoi`, `mon`. The page loads students via `GET /api/sinh-vien?khoa=…&daiDoi=…`.
2. For each student, the page reads `student.diem.find(d => d.mon === selectedMon)` to pre-populate the row.
3. User edits values, clicks **Save** on a row (or **Save all**).
4. **Save on a row that has no existing diem for this `mon`:** `POST /api/diem` with `{ sinhVien, mon, thuongXuyen, mieng?, giuaHP, hetHP }`. Backend pushes a new entry into `SinhVien.diem[]`. The `pre('validate')` hook computes `tbMon`.
5. **Save on a row with existing diem:** `PATCH /api/diem/:id` with the changed fields. Backend updates the matching subdocument.
6. Response includes the freshly computed `tbMon`. Frontend updates the table cell.

### Bulk paste / Excel import

The same tab supports clicking **Nhập từ Excel**, which calls `POST /api/diem/import` with a multipart upload (`file`, `he`, `mon`, `khoa`, `donViLienKet`). See [excel-pipeline.md](../architecture/excel-pipeline.md#5-the-grade-import) for the full contract.

### Teacher restrictions

A teacher can only write grades for students inside their `teacherScope`. The service-layer `applyTeacherScope` + `isPairInTeacherScope` checks enforce this — a write that targets a student outside scope returns 403 even if the route's `authorize(...)` accepts the role.

## Tab 2 — Tổng kết cuối khóa

End-of-cohort summary. For a selected `khoa` (and optionally `daiDoi`, `truong`), shows one row per student with their average across all subjects in the chosen `he`.

### Behavior

1. User selects filters.
2. Frontend calls `GET /api/diem/summary?he=Đại học&khoa=…&daiDoi=…&truong=…`.
3. Backend service builds an aggregation over `SinhVien.diem[]`, computes an overall average across the in-`he` subjects, and returns one record per student.
4. Frontend renders the table. Optionally exports to Excel via `GET /api/bieu-mau/export?he=…&danhSach=Sổ điểm…&khoa=…&daiDoi=…&truong=…`.

The summary view is read-only — no in-place editing.

## Side-effects

- Every grade write is a Mongo write to **one** `SinhVien` document. Even bulk imports do per-student writes (with a parallel chunk size of 50).
- `tbMon` is **always** recomputed from the supplied raw scores — you cannot post a custom `tbMon`. Setting `tbMon` directly in a `POST/PATCH` body is silently overwritten by the hook.
- Grade updates do **not** touch `SinhVien.trangThai` — failing a course does not automatically suspend a student. That's a separate manual decision.
- **Exempt subjects (`SinhVien.monMienHoc`)** — set via the "Môn miễn học" sub-popup in the student add/edit modal. For each exempt subject, the Nhập điểm row renders dashes in `Thường xuyên / Miệng / Giữa HP / Hết HP / TB môn` and the inputs are non-editable. The diem service (`POST/PATCH /api/diem`, `POST /api/diem/import`) refuses to write grades for an exempt subject — single endpoints return `409 SUBJECT_EXEMPT`; the importer skips the row with a soft error. Exemptions do **not** suppress the cross-subject `diemTB` for other subjects — the average naturally excludes the exempt subject (no `tbMon` was ever written), so a student with 3 graded subjects + 1 exempt averages over the 3 graded ones.
- **`trangThai === 'Học ghép'`** — per-subject `tbMon` is still computed normally and grade entry remains editable, but the **cross-subject `diemTB` summary is suppressed** to `—` everywhere it appears (Tổng kết cuối khóa table, xếp loại tally). The two mechanisms compose independently: a student that is both `Học ghép` AND has `monMienHoc` exemptions will see per-subject dashes for exempt subjects AND a cross-subject `—` from the status.

## Failure modes

| Scenario | Result | Recovery |
|---|---|---|
| Raw score out of range (e.g. `10.5`) | `400 VALIDATION_ERROR` from Joi schema (`min: 0, max: 10`) | Fix and retry. |
| Save races: two staff edit the same `(student, mon)` cell | Last-write-wins; one user's edit is silently lost. | Coordinate or refresh before saving. |
| Excel import row references a `maSV` that isn't in the chosen `(khoa, donViLienKet)` scope | Row lands in `missingStudents`, not `inserted`/`updated`. | Add the student first or fix the maSV. |
| Excel import has out-of-range value in a cell | Row contributes a `errors` entry with a Vietnamese reason; other rows still apply. | Fix the cell and re-import. |
| `tbMon` looks wrong in legacy data | Likely written before the integer-arithmetic hook was added. | Run `node backend/src/scripts/fix-tbMon-precision.js` to re-save and recompute. |

## Manual test recipe

- [ ] Pick a `(khoa, daiDoi)` with students. Open Nhập điểm.
- [ ] For one student, enter `(thuongXuyen=8, giuaHP=7, hetHP=9)` for `Quân sự chung` (a Đại học subject). Save. Verify `tbMon = 8.3` per the formula `(8×1 + 7×3 + 9×6)/10 = (8+21+54)/10 = 83/10`.
- [ ] Open `mongosh`, query the SinhVien doc. Confirm `diem[…].tbMon = 8.3` (not `8.299999…`).
- [ ] Edit the row: change `hetHP` to `10`. Save. Confirm `tbMon` recomputes to `(8+21+60)/10 = 8.9`.
- [ ] Go to Tổng kết cuối khóa, select the same khoa. Confirm the student appears with an overall average.
- [ ] Export Excel via Biểu mẫu & in ấn → "Sổ điểm từng môn". Confirm the exported workbook shows the same `tbMon` values.
- [ ] As a teacher with `teacherScope: [{ khoa: <chosen>, daiDoi: <chosen> }]`, try to write a grade for a student in a different daiDoi. Expect 403.
