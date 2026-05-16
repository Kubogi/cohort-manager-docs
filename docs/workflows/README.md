# Workflows

These docs explain how major functions in the system work **end-to-end** — UI → API → service → DB → response → UI update. They're process docs, not reference docs. When you need to know "what does the system actually do when a user clicks X?", start here. When you need "what is the contract of endpoint Y?", go to [`/docs/api/`](../api/) instead.

Each workflow follows the template documented at the bottom of this index.

## By domain

### Authentication & accounts

| Workflow | Trigger | Doc |
|---|---|---|
| Login + session lifecycle | Login page, every protected request | [login-and-session.md](login-and-session.md) |
| User management | Admin creates / edits / disables a user (with teacher scope) | [user-management.md](user-management.md) |

### Students

| Workflow | Trigger | Doc |
|---|---|---|
| Student lifecycle | Create → assign khoa/đại đội → grades → decisions → graduate/withdraw | [student-lifecycle.md](student-lifecycle.md) |
| Decision processing | Admin marks a QĐ/CV as processed/unprocessed; student status updates | [decision-processing.md](decision-processing.md) |
| Auto-battalion assignment | "Biên chế đại đội tự động" rule-based bulk assignment | [auto-battalion-assignment.md](auto-battalion-assignment.md) |
| Excel import of students | Admin uploads a multi-sheet roster `.xlsx` | [excel-import-students.md](excel-import-students.md) |

### Grades & reports

| Workflow | Trigger | Doc |
|---|---|---|
| Grade entry & summary | Per-subject scoring → tbMon auto-compute → end-of-cohort summary | [grade-entry-and-summary.md](grade-entry-and-summary.md) |
| Excel export of forms | "Biểu mẫu & in ấn" — `(he, danhSach, mon)` → templated `.xlsx` | [excel-export-forms.md](excel-export-forms.md) |

### Other

| Workflow | Trigger | Doc |
|---|---|---|
| Health record tracking | Create → status (Bình thường/Viện) → discharge → stats | [health-record-tracking.md](health-record-tracking.md) |
| Learning materials | Folder/file CRUD across 3 drives, quota enforcement | [learning-materials.md](learning-materials.md) |
| Quality survey | Annual link config → GV self-eval + SV feedback → result aggregation | [quality-survey.md](quality-survey.md) |
| Link management | Training-schedule + per-year survey + per-instructor feedback URLs | [link-management.md](link-management.md) |

## Workflow doc template

```
# <Workflow name>

**Triggered from**: <page(s) in docs/frontend/pages/...>
**Touches**: <list of API endpoints, models, services involved>
**Who can do this**: <roles>

## Goal
1-paragraph statement of the user-facing outcome.

## Happy path
Numbered steps from user action → frontend handler → API call → backend service →
DB mutation → response → UI update.

## State diagram   ← only when there are status transitions
Mermaid state machine.

## Side-effects
Cascades, status reverts, file writes, audit fields.

## Failure modes
Validation errors, race conditions, partial-success cases.

## Manual test recipe
Bulleted checklist mirroring the existing TESTING_GUIDE_*.md style.
```
