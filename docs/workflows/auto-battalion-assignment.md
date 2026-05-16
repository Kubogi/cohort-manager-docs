# Auto Battalion Assignment

**Triggered from**: [Biên chế đại đội tự động](../frontend/pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md).

**Touches**: `GET /api/sinh-vien`, `GET /api/don-vi/dai-doi`, `PATCH /api/sinh-vien/:id` (one per moved student).

**Who can do this**: `admin`, `staff`. Not visible to `teacher`.

## Goal

Place students into battalions (đại đội) in bulk. Two flavors:

1. **Initial assignment (Tab 1)** — fresh students with no `daiDoi` get distributed across available battalions in the same `khoa` by some rule (round-robin, fill-first, gender quota, etc. — implementation-specific).
2. **Bulk transfer (Tab 2)** — move a checked set of students from one battalion to another (same `khoa`).

## Happy path — initial assignment

1. Run the [Excel student import](excel-import-students.md) first so the cohort exists.
2. Open Tab 1, pick `khoa` (and optionally `truong`).
3. The page loads:
   - Left pool: students in this khoa with `daiDoi == null` (or marked deprecated).
   - Right pool: `DaiDoi` rows in the khoa, with current `quanSo` and a capacity hint.
4. Pick a distribution rule. Click **Tính toán** — UI computes a preview mapping each unassigned student to a target daiDoi.
5. Inspect the preview. If satisfactory, click **Áp dụng**.
6. Frontend dispatches one `PATCH /api/sinh-vien/:id` per moved student with the new `daiDoi`.

## Happy path — bulk transfer

1. Open Tab 2. Pick source `(khoa, daiDoi)` and destination `daiDoi`.
2. Check the students to move.
3. **Chuyển**. Confirmation modal lists the moves. On confirm, frontend dispatches per-student PATCHes.

## Side-effects

- Each student's `daiDoi` changes; no other fields are touched.
- `DaiDoi.quanSo` is **not** auto-updated. If your operational dashboards rely on `quanSo`, reconcile after a large move.
- Cross-khoa moves are rejected at the validator (`daiDoi.khoa` must match `payload.khoa`).

## Failure modes

| Scenario | Result |
|---|---|
| Network blip mid-batch | Some students moved, some not. UI should show "X of Y succeeded" and let you retry the remainder. |
| Target daiDoi not in the chosen khoa | `400 VALIDATION_ERROR` per affected student. |
| Race with another admin editing the same students | Last write wins. |

## Manual test recipe

- [ ] Import 30 students with no `daiDoi` (use a workbook with a "SỐ LƯỢNG"-only sheet, or just delete `daiDoi` on existing rows).
- [ ] Open Tab 1, pick the khoa, compute preview. Confirm proportional distribution.
- [ ] Apply. Verify all 30 students have a `daiDoi` afterwards.
- [ ] On Tab 2, transfer 5 of them to another daiDoi. Verify the moves stuck in mongosh.
- [ ] Try moving a student to a daiDoi in a different khoa via direct PATCH — expect 400.
