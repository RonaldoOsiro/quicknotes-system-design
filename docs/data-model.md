# QuickNotes Data Model Specification

## 1. Entities & Schema

* **`users`**
  * `id`: `VARCHAR(36)` (Primary Key, UUID)
  * `email`: `VARCHAR(255)` (Unique, Not Null)
  * `password_hash`: `VARCHAR(255)` (Not Null)
  * `created_at`: `TIMESTAMP WITH TIME ZONE` (Default: Current Timestamp)

* **`notes`**
  * `id`: `VARCHAR(36)` (Primary Key, UUID)
  * `user_id`: `VARCHAR(36)` (Foreign Key references `users(id)`, Not Null)
  * `title`: `VARCHAR(100)` (Not Null)
  * `body`: `TEXT` (Nullable)
  * `created_at`: `TIMESTAMP WITH TIME ZONE` (Default: Current Timestamp)
  * `updated_at`: `TIMESTAMP WITH TIME ZONE` (Default: Current Timestamp)

* **`tags`**
  * `id`: `BIGSERIAL` (Primary Key)
  * `user_id`: `VARCHAR(36)` (Foreign Key references `users(id)`, Not Null)
  * `name`: `VARCHAR(50)` (Not Null)

* **`note_tags` (Junction Table)**
  * `note_id`: `VARCHAR(36)` (Foreign Key references `notes(id)`, Not Null)
  * `tag_id`: `BIGINT` (Foreign Key references `tags(id)`, Not Null)
  * *Composite Primary Key*: `(note_id, tag_id)`

---

## 2. Relationships Explanation

* **One-to-Many (`users` -> `notes`)**: One user can create and own multiple notes, but each individual note belongs to exactly one user. Enforced via the `user_id` foreign key in the `notes` table.
* **One-to-Many (`users` -> `tags`)**: One user can create multiple unique tags, but each tag belongs to a single user. Enforced via the `user_id` foreign key in the `tags` table.
* **Many-to-Many (`notes` <-> `tags`)**: A single note can have multiple tags attached to it, and a single tag can be applied to many different notes. This is modeled using the `note_tags` junction table.

---

## 3. SQL `CREATE TABLE` Statements

```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notes (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    title VARCHAR(100) NOT NULL,
    body TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_notes_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    name VARCHAR(50) NOT NULL,
    CONSTRAINT fk_tags_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE note_tags (
    note_id VARCHAR(36) NOT NULL,
    tag_id BIGINT NOT NULL,
    PRIMARY KEY (note_id, tag_id),
    CONSTRAINT fk_notetags_note FOREIGN KEY (note_id) REFERENCES notes(id) ON DELETE CASCADE,
    CONSTRAINT fk_notetags_tag FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);
```

## 4. Common SQL Queries

The following examples use PostgreSQL and assume the authenticated user's ID is supplied as a parameter.

### 4.1 List a user's notes with their tags

This query returns a page of notes, newest first, with each note's tags aggregated into a JSON array. Notes without tags are retained with an empty array.

```sql
SELECT
    n.id,
    n.title,
    n.body,
    n.created_at,
    n.updated_at,
    COALESCE(
        jsonb_agg(jsonb_build_object('id', t.id, 'name', t.name))
            FILTER (WHERE t.id IS NOT NULL),
        '[]'::jsonb
    ) AS tags
FROM notes AS n
LEFT JOIN note_tags AS nt ON nt.note_id = n.id
LEFT JOIN tags AS t ON t.id = nt.tag_id AND t.user_id = n.user_id
WHERE n.user_id = $1
GROUP BY n.id
ORDER BY n.created_at DESC, n.id
LIMIT $2 OFFSET $3;
```

### 4.2 Retrieve one note and its tags

The owner condition ensures that a user cannot retrieve another user's note by guessing its ID.

```sql
SELECT
    n.id,
    n.title,
    n.body,
    n.created_at,
    n.updated_at,
    COALESCE(
        jsonb_agg(jsonb_build_object('id', t.id, 'name', t.name))
            FILTER (WHERE t.id IS NOT NULL),
        '[]'::jsonb
    ) AS tags
FROM notes AS n
LEFT JOIN note_tags AS nt ON nt.note_id = n.id
LEFT JOIN tags AS t ON t.id = nt.tag_id AND t.user_id = n.user_id
WHERE n.id = $1
  AND n.user_id = $2
GROUP BY n.id;
```

### 4.3 Find a user's notes by tag

This query returns a paginated list of notes belonging to the user and associated with the requested tag.

```sql
SELECT n.id, n.title, n.body, n.created_at, n.updated_at
FROM notes AS n
JOIN note_tags AS nt ON nt.note_id = n.id
JOIN tags AS t ON t.id = nt.tag_id
WHERE n.user_id = $1
  AND t.user_id = $1
  AND t.id = $2
ORDER BY n.created_at DESC, n.id
LIMIT $3 OFFSET $4;
```

## 5. Indexing Strategy

```sql
CREATE INDEX idx_notes_user_created
    ON notes (user_id, created_at DESC, id);

CREATE INDEX idx_note_tags_tag_note
    ON note_tags (tag_id, note_id);
```

The `notes` index supports the common per-user listing and ordering by creation time, reducing the rows scanned and avoiding a separate sort for paginated results. The `note_tags` primary key already indexes `(note_id, tag_id)` for note-to-tag lookups; the additional reverse-order index supports finding notes by tag. The unique constraint on `users.email` already creates an index for account lookup. Add full-text indexes for title/body search only when query patterns and language configuration are established; a general-purpose index on the large `body` text column is not an effective search strategy.

## 6. Why a Relational Database

PostgreSQL fits QuickNotes because users, notes, and tags have clear relationships, and note/tag assignments require a many-to-many junction table. Foreign keys and transactions enforce ownership and referential integrity, while a single transaction can create a note and attach its tags consistently. SQL also supports the filtering, pagination, and joins needed by the API, and PostgreSQL provides mature indexing, replication, backups, and full-text search. A document store could embed tags in each note, but shared tags and tag-based lookups would introduce duplication or additional consistency work; the relational model is a better match for these access patterns and constraints.