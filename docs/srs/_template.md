# 🧩 MODULE: <NAME>

## 1. Purpose

<One paragraph. What does this module do, for whom, and why does it exist?>

## 2. Scope

**In Scope**

- ...

**Out of Scope**

- ...

## 3. Actors

- Admin
- ...
- System

## 4. Core Concepts

### <Concept name>

<Definition. Use this section to establish module-specific vocabulary so FRs and data models can use the terms without re-explaining.>

## 5. Functional Requirements

### 5.1 <Sub-area name>

- **FR-<XX>-001:** System shall ...
- **FR-<XX>-002:** <Actor> shall ...

### 5.2 <Sub-area name>

- **FR-<XX>-003:** ...

## 6. Business Rules

- ...

## 7. User Flow / Process Flow

### 7.1 <Flow name>

1. ...
2. ...

### 7.2 <Flow name>

1. ...

## 8. Data Model

### `<EntityName>`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| created_at | Timestamp | Created time |
| updated_at | Timestamp | Updated time |

### `<EntityName>`

| Field | Type | Description |
|---|---|---|
| ... | ... | ... |

## 9. API Contracts

### `<Endpoint label>`

```http
POST /<path>
```

**Headers**

```
Authorization: Bearer <token>
```

**Request**

```json
{
  "field": "value"
}
```

**Response (200)**

```json
{
  "field": "value"
}
```

**Errors**

- `400` — ...
- `401` — ...
- `403` — ...
- `404` — ...

## 10. UI Components

### Screen: <name>

- Components: ...
- Actions: ...
- States: ...

## 11. Validation Rules

- ...

## 12. Edge Cases

- Network failure during ...
- Concurrent edit ...
- Deleted referenced entity ...
- Permission changed mid-session ...
- ...

## 13. Dependencies

- <Other module> — for ...

## 14. Non-Functional Requirements

- Performance: ...
- Accessibility: ...
- Scale: ...

## 15. Assumptions

- ...

## 16. Open Questions

- ...
