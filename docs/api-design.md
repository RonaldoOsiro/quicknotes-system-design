# QuickNotes REST API Design

## Base URL
`https://api.quicknotes.com/v1`

---

## REST Endpoints

| Method | Path | Description | Success Status |
| :--- | :--- | :--- | :--- |
| `GET` | `/notes` | Retrieve a paginated list of notes for the authenticated user | `200 OK` |
| `POST` | `/notes` | Create a new note | `201 Created` |
| `GET` | `/notes/{id}` | Retrieve a specific note by ID | `200 OK` |
| `PUT` | `/notes/{id}` | Fully update an existing note | `200 OK` |
| `DELETE` | `/notes/{id}` | Delete a note by ID | `204 No Content` |
| `GET` | `/tags` | List all tags created by the authenticated user | `200 OK` |

---

## Request & Response Examples

### 1. Creating a Note (`POST /notes`)
**Request Body:**
```json
{
  "title": "System Architecture Review",
  "body": "Prepare slides for the engineering team sync on Tuesday morning.",
  "tag_ids": [1, 4]
}
```

**Response (`201 Created`):**
```json
{
  "id": "a7c1a651-0f39-4e40-8731-b1e9ccdbb2f2",
  "user_id": "7b4d00a2-3bd1-48b2-9f2c-6862f79b8a10",
  "title": "System Architecture Review",
  "body": "Prepare slides for the engineering team sync on Tuesday morning.",
  "tag_ids": [1, 4],
  "created_at": "2026-10-24T19:30:00Z",
  "updated_at": "2026-10-24T19:30:00Z"
}
```

### 2. Listing Notes (`GET /notes`)
Notes are returned newest first. `page` is one-based; `limit` is the maximum number of notes per page. If omitted, `page` defaults to `1` and `limit` to `20`.

**Request:**
```http
GET /v1/notes?page=1&limit=20
Authorization: Bearer <access_token>
Accept: application/json
```

**Response (`200 OK`):**
```json
{
  "data": [
    {
      "id": "a7c1a651-0f39-4e40-8731-b1e9ccdbb2f2",
      "user_id": "7b4d00a2-3bd1-48b2-9f2c-6862f79b8a10",
      "title": "System Architecture Review",
      "body": "Prepare slides for the engineering team sync on Tuesday morning.",
      "tag_ids": [1, 4],
      "created_at": "2026-10-24T19:30:00Z",
      "updated_at": "2026-10-24T19:30:00Z"
    },
    {
      "id": "c92f1873-e172-45ee-b297-151621a38051",
      "user_id": "7b4d00a2-3bd1-48b2-9f2c-6862f79b8a10",
      "title": "Review meeting notes",
      "body": "Summarize action items from the project meeting.",
      "tag_ids": [],
      "created_at": "2026-10-23T15:00:00Z",
      "updated_at": "2026-10-23T15:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 2,
    "total_pages": 1
  }
}
```

An empty page returns `"data": []` and the corresponding pagination metadata.

## Error Responses

All errors use the same JSON envelope. `details` is an array of field-level issues when applicable and is otherwise empty.

| Field | Type | Description |
| :--- | :--- | :--- |
| `error.code` | `string` | Stable machine-readable error code. |
| `error.message` | `string` | Human-readable explanation of the error. |
| `error.details` | `array` | Optional field-level issues; each item contains a `field` and an `issue`. Empty when no field-specific details apply. |

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request is invalid.",
    "details": [
      {
        "field": "title",
        "issue": "Title is required."
      }
    ]
  }
}
```

| Status | Code | Meaning |
| :--- | :--- | :--- |
| `400 Bad Request` | `VALIDATION_ERROR` | The request is malformed or a field fails validation. |
| `401 Unauthorized` | `UNAUTHENTICATED` | Authentication is missing, invalid, or expired. |
| `403 Forbidden` | `FORBIDDEN` | The authenticated user is not permitted to perform this action. |
| `404 Not Found` | `NOT_FOUND` | The requested resource does not exist or is not available to the user. |
| `500 Internal Server Error` | `INTERNAL_ERROR` | An unexpected server error occurred. |

For example, an unauthenticated request returns `401 Unauthorized`:
```json
{
  "error": {
    "code": "UNAUTHENTICATED",
    "message": "Authentication is required.",
    "details": []
  }
}
```