# GET /api/ho-so-suc-khoe/stats

**Endpoint**: `GET /api/ho-so-suc-khoe/stats`

**Authentication**: ✅ Required

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

**Last Verified**: 2026-05-16

---

## Description

Returns a breakdown of health records by `trangThai` (status). Each element in the response array represents one status value and the count of records with that status.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

None. The service accepts no query parameters — all filtering is handled by unit scoping (user's `allowedUnits` for staff, `teacherScope` for teacher). Passing filter parameters has **no effect**.

---

## Response

### Success (200 OK)

```json
{
  "data": [
    { "trangThai": "Bình thường", "soCa": 65 },
    { "trangThai": "Viện", "soCa": 13 }
  ]
}
```

Each element:

| Field | Type | Description |
|-------|------|-------------|
| trangThai | string | Status value. Known values: `"Bình thường"`, `"Viện"`. `null` values in the collection are mapped to `"Khac"`. |
| soCa | number | Count of records with this status. |

---

## Notes

- There is no total count, no `byKhung`, no `byBenhVien`, and no `recentVisits` in the response — the service only aggregates by `trangThai`.
- Unit scoping is applied to the `$match` stage; staff see only records for students in their `allowedUnits`, teachers see only records within their `teacherScope`.

---

## Related

- [GET /api/ho-so-suc-khoe](./get-list.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
