# POST /api/bien-che/partition

**Endpoint**: `POST /api/bien-che/partition`
**Authentication**: ✅ Required
**Roles**: admin, staff
**Unit Scoping**: ❌ Not applied (pure compute, no DB read/write)
**Last Verified**: 2026-05-23

---

## Description

Partitions a user-supplied list of classes into `soDaiDoi` buckets so the per-bucket student totals are as balanced as possible. Pure planning tool — no records are read from or written to the database. Backs the "Biên chế đại đội tự động" page.

Algorithm: greedy **LPT (Longest Processing Time first)** seed followed by a **pairwise swap + move refinement** loop until no further improvement is possible.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
Content-Type: application/json
```

### Body

```json
{
  "classes": [
    { "ten": "Lớp 1", "soSinhVien": 57 },
    { "ten": "Lớp 2", "soSinhVien": 68 }
  ],
  "soDaiDoi": 3
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `classes` | `{ ten: string, soSinhVien: number }[]` | Yes | Non-empty list of classes. `ten` must be a non-empty trimmed string and unique across the list. `soSinhVien` must be a positive integer. |
| `soDaiDoi` | number | Yes | Positive integer, must be `≤ classes.length`. Stringy JSON values (e.g. `"3"`) are coerced. |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "daiDois": [
      {
        "stt": 1,
        "classes": [
          { "ten": "Lớp 5", "soSinhVien": 74 },
          { "ten": "Lớp 3", "soSinhVien": 70 }
        ],
        "total": 144
      },
      {
        "stt": 2,
        "classes": [
          { "ten": "Lớp 4", "soSinhVien": 64 },
          { "ten": "Lớp 1", "soSinhVien": 57 },
          { "ten": "Lớp 6", "soSinhVien": 27 }
        ],
        "total": 148
      },
      {
        "stt": 3,
        "classes": [
          { "ten": "Lớp 2", "soSinhVien": 68 },
          { "ten": "Lớp 7", "soSinhVien": 43 },
          { "ten": "Lớp 8", "soSinhVien": 31 }
        ],
        "total": 142
      }
    ],
    "max": 148,
    "min": 142,
    "spread": 6
  }
}
```

- `daiDois[*].stt` is 1-indexed.
- `daiDois[*].classes` is sorted by `soSinhVien` desc (alphabetic on `ten` for ties) for display.
- `spread = max - min`. Zero means perfect balance.

---

## Error Responses

### 400 Bad Request — validation
Returned with `{ code: 'INVALID_INPUT' }` and a Vietnamese `message`. Triggers:

- Empty / non-array `classes`.
- Class with non-positive or non-integer `soSinhVien`.
- Empty / missing `ten` (after trim).
- Duplicate `ten` (after trim) — duplicates are rejected to keep the result unambiguous.
- `soDaiDoi` non-integer or `< 1`.
- `soDaiDoi > classes.length` — message: "Số đại đội không được lớn hơn số lớp."

### 401 Unauthorized
Missing or invalid bearer token.

### 403 Forbidden
Role is not `admin` or `staff` (e.g. `viewer`, `teacher`).

---

## Notes

- Algorithm is deterministic: equal inputs produce equal output (LPT tiebreaks on alphabetic `ten`; swap pass picks the strictly-best move and breaks pair ties by lower bucket index).
- The endpoint never touches MongoDB. It does not modify existing `DaiDoi` records, student `daiDoi` assignments, or anything else — the result is informational only. To actually move students, use the existing `PATCH /api/sinh-vien/:id` flow.
- Practical scale: tested at a few dozen classes × ~10 buckets. The swap pass is O(k² · n²) per iteration, so don't feed thousands of classes.

## Related

- [Biên chế đại đội tự động page](../../../frontend/pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md)
- Source: `backend/src/utils/partitionClasses.js`
