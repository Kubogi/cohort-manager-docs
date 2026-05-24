# DELETE /api/ho-so-suc-khoe/:id

**Endpoint**: `DELETE /api/ho-so-suc-khoe/:id`  
**Authentication**: ✅ Required  
**Roles**: admin
**Last Verified**: 2026-05-23

---

## Description

Deletes a health record. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | HoSoSucKhoe MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found
Health record doesn't exist.

### 403 Forbidden
Only admin role can delete health records.

---

## Notes

- Returns 204 (No Content) on success
- Does NOT delete referenced SinhVien document
- **Side effect**: re-syncs `SinhVien.ghiChuYTe` for the owning student. The discharge-history block (`"Đi viện lần N: dd/mm"` lines) is rebuilt from the surviving HoSoSucKhoe records, so deleting the first record renumbers the remaining lines correctly (lần 2 becomes lần 1). User-authored notes that don't match the discharge-line pattern are preserved verbatim. The re-sync runs only after the deletion succeeds. See `backend/src/utils/ghiChuYTeSync.js`.

---

## Related

- [GET /api/ho-so-suc-khoe/:id](./get-one.md)
- [PATCH /api/ho-so-suc-khoe/:id](./patch.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
