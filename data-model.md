# URL Shortener - Data Model Documentation

In this document, we define the data models used in the system, their relationships, and how we generate the `short_code`.

## Data Models

The system contains three main entities:

1. `User`
2. `URL`
3. `UserURL`

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

The `urls` table stores globally shortened URLs. URL identity is global, so the same `original_url` always maps to the same `short_code` across the entire system.

| Field | Type | Required? | Purpose |
|---|---|---|---|
| `id` | BIGINT | Yes | Primary key and internal URL identifier |
| `original_url` | TEXT | Yes | Stores the original URL for redirection |
| `short_code` | VARCHAR(7) | Yes | Short code used in the short URL |
| `created_at` | DateTime | Yes | Tracks when the global URL record was created |

### Relationships

- Each `urls` row represents one unique original URL.
- User ownership is stored in `user_urls`, not directly on `urls`.

```text
users (1) ---- (N) user_urls (N) ---- (1) urls
```

### Constraints

- `id` is the primary key.
- `original_url` must be unique.
- `short_code` must be unique.
- `original_url` must contain a valid URL.
- `short_code` is generated with the cryptographically secure random generation strategy described below.

### Indexes

- Unique index on `original_url` for global URL deduplication.
- Unique index on `short_code` for redirect lookups.

---

## 3. User URLs Table

The `user_urls` table stores which users own or have saved which global URL records.

| Field | Type | Required? | Purpose |
|---|---|---|---|
| `user_id` | BIGINT | Yes | References the user who owns the URL association |
| `url_id` | BIGINT | Yes | References the global URL record |
| `created_at` | DateTime | Yes | Tracks when the user added the URL |
| `deleted_at` | DateTime | No | Used for soft deletion of the user's ownership association |

### Relationships

- `user_id` is a foreign key referencing `users.id`.
- `url_id` is a foreign key referencing `urls.id`.
- A user can own many URLs.
- A global URL can belong to many users.

```text
User A --+
         +-- https://example.com -> a8Kp2Qz
User B --+
```

### Constraints

- A user must not have multiple active associations with the same URL.
- Because `user_urls` supports soft deletion, PostgreSQL should enforce this with a partial unique index on active rows:

```sql
CREATE UNIQUE INDEX user_urls_active_unique
ON user_urls (user_id, url_id)
WHERE deleted_at IS NULL;
```

This keeps the existing soft-delete behavior: a deleted association remains for history, while the same user can later add the same URL again by creating a new active association.

### Indexes

- Partial unique index on `(user_id, url_id)` where `deleted_at IS NULL`.
- Index on `(user_id, created_at)` for retrieving a user's active URLs with pagination.
- Index on `url_id` if ownership lookups by global URL are needed.

---

## Redis Data

Redis stores temporary data that does not belong in PostgreSQL.

### Sessions

After a successful login, the API Server generates a random `session_id`.

The `session_id` is stored in Redis and mapped to the authenticated `user_id`.
The client receives the `session_id` in a secure `HttpOnly` cookie and automatically sends it with authenticated requests.

Sessions must have a TTL so they expire automatically.

Example:

```text
session:{session_id} -> user_id
TTL: configured session lifetime
```

Session IDs are not stored in PostgreSQL.

### Rate-limiting State

Redis also stores rate-limiting state.

For authenticated URL creation, the rate limit is based on `user_id`.

Example:

```text
rate_limit:create_url:{user_id} -> request count or timestamp
TTL: rate-limit window
```

Public short-URL redirects do not require authentication and are not subject to this authenticated user-based rate limiter.

## How to Generate `short_code`

Short codes are generated directly with a cryptographically secure random generator. Each code initially contains 7 characters selected from `a-z`, `A-Z`, and `0-9`.

The generated code is independent of `urls.id`; the `BIGINT` primary key remains an internal database identifier and is not Base62-encoded. This avoids predictable short codes derived from sequential IDs.

The `short_code` column has a unique constraint. The application inserts the generated code and retries with a newly generated code if PostgreSQL reports a unique-constraint collision. It does not run a separate existence query before insertion, since the constraint handles the race safely.

Short-code generation only happens when the submitted `original_url` does not already exist in the global `urls` table. If the `original_url` already exists, the system reuses the existing `short_code`.

---

## Global URL Deduplication

The same original URL should always map to the same short URL across the entire system.

```text
User A submits https://example.com -> a8Kp2Qz
User B submits https://example.com -> a8Kp2Qz
```

The `urls.original_url` unique constraint enforces this rule. On URL creation, the system first looks for an existing global URL by `original_url`. If it exists, the system creates or restores the user's active `user_urls` association and returns the existing short URL. If it does not exist, the system creates a new `urls` row with a new secure random `short_code`, then creates the user's `user_urls` association.

Concurrent requests for the same new `original_url` must rely on the `urls.original_url` unique constraint. If one request inserts first and another receives a unique-constraint conflict, the second request reads the existing row and creates the user's association to that row.

---

## Soft Deletion

The system uses soft deletion for user ownership associations instead of permanently removing those records.

When a user deletes a URL from their account:

```text
user_urls.deleted_at = current_timestamp
```

The global `urls` row remains in the database because other users may still own the same shortened URL and public redirects should continue to work.

### Benefits

- Maintains ownership history.
- Allows auditing.
- Prevents accidental permanent data loss.
- Makes recovery possible if required.

### Query Behavior

When retrieving a user's URLs:

```sql
SELECT urls.*
FROM user_urls
JOIN urls ON urls.id = user_urls.url_id
WHERE user_urls.user_id = ?
  AND user_urls.deleted_at IS NULL
ORDER BY user_urls.created_at DESC
```

When redirecting a short URL:

```sql
WHERE short_code = ?
```

Redirect behavior depends on the global `urls` row, not on a specific user's ownership association. A user deleting their association must not break redirects for the same short URL.

---

## Summary

The system uses three main tables:

- `users` - Stores user accounts and securely hashed passwords.
- `urls` - Stores globally unique original URLs and their short codes.
- `user_urls` - Stores per-user ownership associations with soft deletion.

The `urls` table does not contain `user_id`. A many-to-many relationship between users and URLs is represented through `user_urls`.

Redis stores temporary session data and rate-limiting state. PostgreSQL remains the persistent storage for users, URLs, and ownership associations, and session IDs are not stored in PostgreSQL.

The `short_code` is generated randomly and securely only for new global URLs, while PostgreSQL enforces uniqueness for both `original_url` and `short_code`. The `urls.id` field remains a BIGINT primary key but is not used to generate the short code.

# What Is Next?

Continue -> [High-Level Architecture](high-level-architecture.md)
