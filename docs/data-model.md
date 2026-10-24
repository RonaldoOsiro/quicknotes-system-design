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