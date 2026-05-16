# API Contracts: <Module>

This file defines the REST endpoints for `<module>`. Frontend and backend share this contract — no endpoint exists unless it's documented here.

## Conventions

- Base path: `/api/v1/`
- Auth: `Authorization: Bearer <jwt>` unless explicitly marked public
- All errors return `{ "code": "<ERROR_CODE>", "message": "<human readable>", "details": {} }`
- Pagination: `?page=1&pageSize=20`; response includes `{ items, total, page, pageSize }`
- Timestamps: ISO 8601 with timezone (`2026-05-15T09:00:00+06:00`)
- Money: `{ "amount": "3500.00", "currency": "BDT" }` (amount as string to preserve decimal precision)

## Endpoints

### <Endpoint label>

**Satisfies:** FR-XX-001, FR-XX-002

```http
POST /api/v1/<resource>
```

**Headers**

```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request body**

```json
{
  "field": "value"
}
```

**Response — 201 Created**

```json
{
  "id": "uuid",
  "field": "value",
  "createdAt": "2026-05-15T09:00:00+06:00"
}
```

**Errors**

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body invalid |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Role lacks permission |
| 409 | `CONFLICT` | Duplicate resource |

### <Next endpoint>

...
