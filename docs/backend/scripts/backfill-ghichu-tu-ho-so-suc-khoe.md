# `backfill-ghichu-tu-ho-so-suc-khoe.js`

**Source**: [backend/src/scripts/backfill-ghichu-tu-ho-so-suc-khoe.js](../../../backend/src/scripts/backfill-ghichu-tu-ho-so-suc-khoe.js)

## What it does

For every `HoSoSucKhoe` document with a valid `sinhVien` and `thoiGianDi.ngay`, append a line to the referenced student's `ghiChuYTe`:

```
Đi viện lần <N>: <dd>/<mm>
```

`<N>` is the admission order for that student (sorted by `thoiGianDi.ngay` ascending). `<dd>/<mm>` is the admission day and month.

This mirrors the live UI's `appendDischargeNote` so manually-imported health records (those that bypass the frontend) end up with the same Ghi chú coverage as records created through Hồ sơ sức khỏe → Sửa → discharge.

## When to run

- After deploying any HoSoSucKhoe data dump that landed in MongoDB outside the UI.
- Optionally any time you suspect drift between `HoSoSucKhoe` documents and the per-student Ghi chú list. The script is idempotent — re-running adds nothing new.

## How to run

```bash
cd backend
npm run backfill:ghichu-suc-khoe -- --dry-run   # preview what would change
npm run backfill:ghichu-suc-khoe                # apply the changes
```

The dry-run prints the per-student diff (students affected, lines being added) without writing. Re-run without `--dry-run` to apply.

## Behaviour notes

- **Idempotent**: existing lines are detected via substring match against the student's current `ghiChuYTe`; only missing lines are appended (joined by newline).
- **Records without `thoiGianDi.ngay` are skipped.** They don't count toward the numbering — slot N goes to the Nth record with a valid date.
- **Orphan records** (pointing at a student that doesn't exist anymore) are silently skipped — no error, no warning. Cleanup of orphans is out of scope for this script.
- **Existing manually-edited notes** are preserved verbatim; the script only appends.

## Related

- [`appendDischargeNote`](../../../frontend/src/resources/quanLySinhVien/hoSoSucKhoe/useHoSoSucKhoeData.ts) — the live equivalent invoked from the Hồ sơ sức khỏe modal.
- [`HoSoSucKhoe` schema](../schemas/HoSoSucKhoe.md)
