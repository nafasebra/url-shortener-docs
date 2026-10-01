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

Redis stores temporary session data and rate-limiting state. PostgreSQL remains the persistent storage for users and URLs, and session IDs are not stored in PostgreSQL.

The `short_code` is generated randomly and securely, while PostgreSQL enforces its uniqueness. The `urls.id` field remains a BIGINT primary key but is not used to generate the short code.

# What Is Next?

Continue → [High-Level Architecture](high-level-architecture.md)
