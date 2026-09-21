# URL Shortener - Data Model Documentation

In this document, we define the data models used in the system, their relationships, and how we generate the `short_code`.

## Data Models

The system contains two main entities:

1. `User`
2. `URL`

---

## 1. Users Table

The `users` table stores registered user information.

| Field | Type | Required? | Purpose |
|---|---|---|---|
| `id` | BIGINT | Yes | Primary key and user identifier |
| `email` | VARCHAR(255) | Yes | User's unique email address |
| `password_hash` | VARCHAR(255) | Yes | Stores the securely hashed password |
| `created_at` | DateTime | Yes | Tracks when the user was created |
| `updated_at` | DateTime | Yes | Tracks the last update |

### Constraints

- `id` is the primary key.
- `email` must be unique.
- Passwords must never be stored in plain text.
- Passwords must be securely hashed using a suitable password-hashing algorithm. 

### Indexes

- Unique index on `email` for efficient login and uniqueness validation.

---

## 2. URLs Table

The `urls` table stores the URLs created by users.

| Field | Type | Required? | Purpose |
|---|---|---|---|
| `id` | BIGINT | Yes | Primary key and internal URL identifier |
| `user_id` | BIGINT | Yes | Identifies the user who created the URL |
| `short_code` | VARCHAR(7) | Yes | Short code used in the short URL |
| `original_url` | TEXT | Yes | Stores the original URL for redirection |
| `created_at` | DateTime | Yes | Used for tracking and pagination |
| `deleted_at` | DateTime | No | Used for soft deletion |

### Relationships

- Each URL belongs to one user.
- A user can create multiple URLs.
- `user_id` is a foreign key referencing `users.id`.

```text
users (1) ──────────── (N) urls
```

### Constraints

- `id` is the primary key.
- `short_code` must be unique.
- `user_id` must reference an existing user.
- `original_url` must contain a valid URL.
- `deleted_at` is `NULL` for active URLs.

### Indexes

- Unique index on `short_code` for redirect lookups.
- Index on `(user_id, created_at)` for retrieving a user's URLs with pagination.
- Index on `(ip_address, created_at)` if needed for rate-limiting queries.

---

## How to Generate `short_code`

Since we already have a unique `BIGINT` ID, we can use **Base62 encoding** to generate the `short_code`.

Base62 uses 62 characters:

- A-Z: 26 characters
- a-z: 26 characters
- 0-9: 10 characters

For example:

```text
ID: 123456789
       ↓
   Base62 Encode
       ↓
short_code: 8M0kX
```

Using the database ID as the source of the `short_code` guarantees uniqueness because the ID is a unique primary key.

### Generation Process

1. Insert the URL record into the database.
2. Retrieve the generated `id`.
3. Convert the ID to Base62.
4. Update the record with the generated `short_code`.

This process should be performed inside a database transaction.

### Advantages

This approach is appropriate for the current stage of the system because it is:

- Simple
- Fast
- Easy to implement
- Deterministic
- Free from collision handling during generation

### Limitations

Generated short codes are predictable and can potentially be enumerated.

For example, if a user knows that a URL has the short code generated from ID `123`, they might be able to guess codes generated from nearby IDs.

If privacy becomes an important requirement, we can switch to a random or non-sequential short-code generation strategy.

In that case:

- Generate a random short code.
- Add a unique constraint on `short_code`.
- Check for collisions.
- Regenerate the code if a collision occurs.

---

## Soft Deletion

The system uses soft deletion instead of permanently removing URL records.

When a URL is deleted:

```text
deleted_at = current_timestamp
```

The record remains in the database, but it is treated as deleted.

### Benefits

- Maintains historical data.
- Allows auditing.
- Prevents accidental permanent data loss.
- Makes recovery possible if required.

### Query Behavior

When retrieving a user's URLs:

```sql
WHERE user_id = ?
  AND deleted_at IS NULL
```

When redirecting a short URL:

```sql
WHERE short_code = ?
  AND deleted_at IS NULL
```

If a short URL exists but has been deleted, the API returns:

```http
410 Gone
```

---

## Summary

The system uses two main tables:

- `users` — Stores user accounts and securely hashed passwords.
- `urls` — Stores URLs created by authenticated users.

The `urls` table references the `users` table through `user_id`.

The `short_code` is generated using Base62 encoding based on the URL's unique database ID.

# What Is Next?

Continue → [High-Level Architecture](high-level-architecture.md)
