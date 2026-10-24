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